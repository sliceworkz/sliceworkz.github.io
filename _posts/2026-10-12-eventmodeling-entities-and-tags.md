---
layout: post
toc: true
title: Entities, Typed Ids and Tags
description: Declaring the kinds of thing your events are about, with an id type per entity and type-safe tags
date: 2026-10-12 02:00:00
categories: [Eventmodeling Documentation,Eventmodeling Guides]
tags: [eventmodeling, entity, entityid, tags, dcb, typed ids]
---

Every event your domain records is *about* something: an account, a customer, a payment, a month. In the Sliceworkz EventModeling framework that something is an **entity**, and the entity is what turns an id into a tag on an event — the tag a read model queries by and a command's consistency boundary is built on. This guide covers declaring entities, the typed ids that go with them, how events are tagged and read back, and what of it is stored data.

## Why Tags, and Why Entities

The [Dynamic Consistency Boundary](/posts/aggregates-and-dcb/) model has no aggregates and no stream per entity: an event is stored once, in its context's stream, and carries **tags** saying what it is about. `account:acc-42` on an event says it concerns account `acc-42`. Reading "everything about account acc-42" is a [query on that tag](/posts/eventstore-querying-events/), and a command deciding on account `acc-42` locks on that same query. An event about two things — a transfer between two accounts, an account opened for a customer — simply carries both tags.

A tag is a key and a value, both strings. Left at that, every command builds its tags by hand: `Tag.of("account", accountId)` in one place, `Tag.of("acount", ...)` in another, a customer id passed where an account id belonged. None of those fails to compile, and each surfaces as a query that finds nothing. An `Entity` gives the domain a vocabulary over tags that the compiler checks.

## Declaring Entities

Declare each kind of entity once, in the domain interface, next to a one-line record for its id:

```java
public interface BankingDomain {

    record AccountId ( String value ) implements EntityId { }
    record CustomerId ( String value ) implements EntityId { }

    Entity<AccountId>  ACCOUNT  = Entity.of("account", AccountId::new);
    Entity<CustomerId> CUSTOMER = Entity.of("customer", CustomerId::new);

    sealed interface BankingDomainEvent {
        record AccountOpened ( AccountId accountId, CustomerId customerId, LocalDate date ) implements BankingDomainEvent { }
        record MoneyDeposited ( AccountId accountId, BigDecimal amount ) implements BankingDomainEvent { }
    }
    // ...
}
```

`Entity.of(name, factory)` takes the entity's name — which becomes the key of its tags — and how to make an id out of a stored string, which is the record's constructor reference. `EntityId` is an interface with one method, `value()`, which the record's component provides.

## One Id Type per Entity

The id record is what the whole mechanism rests on. With one id type per entity:

- `new AccountOpened(customerId, accountId, date)` — arguments swapped — does not compile.
- A read model constructed with an `AccountId` cannot be handed a `CustomerId`.
- `ACCOUNT.tag(customerId)` does not compile: an entity tags only its own ids.

The alternative — one shared id type for every entity, `AccountOpened(Id accountId, Id customerId)` — loses because every one of those mistakes then compiles, and turns up as a query that finds nothing or a tag on the wrong entity, never as an error. Every call site names the entity anyway (`accountId`, `customerId`); the record makes the compiler see what the variable name says. One line per entity is the whole price.

## Minting, Normalising, Tagging

The entity is the only place ids of its kind are made:

| Method | Returns | Use it for |
|---|---|---|
| `ACCOUNT.newId()` | a fresh `AccountId`, a random UUID | minting the id of a new account, inside the command that creates it |
| `ACCOUNT.id(String)` | an `AccountId` | a request parameter, a value read from elsewhere, a fixture in a test |
| `ACCOUNT.tag(id)` | `Tag` `account:<id>` | combining with other tags in `Tags.of(...)` |
| `ACCOUNT.tags(id)` | `Tags` holding that one tag | a query, or an append with a single tag |
| `ACCOUNT.tag()` | `Tag` `account` — the bare key | a flag (below) |
| `ACCOUNT.idIn(tags)` | `Optional<AccountId>` | reading the account an event is about back off its tags |
| `ACCOUNT.idsIn(tags)` | `Set<AccountId>` | reading every account an event is about |
| `ACCOUNT.name()` | `"account"` | the tag key, where you need it as a string |

A command mints and tags:

```java
@Override
public AccountId execute ( CommandContext<BankingDomainEvent, BankingDomainEvent> context ) {

    var result = context.noDecisionModels();

    AccountId accountId = ACCOUNT.newId();

    result.raiseEvent(
        new AccountOpened(accountId, customerId, LocalDate.now()),
        Tags.of(ACCOUNT.tag(accountId), CUSTOMER.tag(customerId)));

    return accountId;
}
```

A read model queries by the same tag:

```java
@Override
public EventQuery eventQuery ( ) {
    return EventQuery.forTags(ACCOUNT.tags(accountId));
}
```

And an endpoint turns a path parameter into an id:

```java
app.read(AccountDetailsReadModel.class, ACCOUNT.id(params.get("accountId")))
```

### Normalisation

`id(String)` strips leading and trailing whitespace, so `ACCOUNT.id("  acc-42 ")` equals `ACCOUNT.id("acc-42")`, and equality and tag matching see one spelling. A null or blank value is an `IllegalArgumentException`: it identifies nothing. `tag(null)` is refused for the same reason — an event cannot be tagged as being about no account.

Use `id(String)` wherever a string becomes an id, rather than calling the record's constructor yourself. `new AccountId(" acc-42")` is a different id from `ACCOUNT.id("acc-42")`, and a query for one will not find events tagged with the other.

### The Id Record Stays Lenient

The id record has no validation of its own, and that is deliberate. An id is carried inside event payloads, and when an event is read back the JSON mapper reconstructs the payload record — and the id records inside it — through their canonical constructors, on every read of history. A constructor that throws turns a rule tightened today into a [poison event](/posts/eventstore-error-handling/) for every event written before it. Rules about what an id may look like belong in the factory, `Entity.id(...)`, and in [the input checks of a command](/posts/eventmodeling-where-validations-go/) — never in the record.

> Keep `record AccountId ( String value ) implements EntityId { }` exactly that: one component, no compact constructor, no validation.
{: .prompt-warning }

## Reading an Id Back Off an Event

Inside a read model's `when`, or an automation's todo list, the tags of an event say what it is about. `idIn` reads one id back:

```java
@Override
public void when ( Event<BankingDomainEvent> event ) {
    switch ( event.data() ) {
        case AccountOpened opened -> {
            CustomerId customer = CUSTOMER.idIn(event.tags()).orElseThrow();
            // ...
        }
        case MoneyDeposited deposited -> { /* ... */ }
    }
}
```

Most of the time the payload carries the id as well — `opened.customerId()` — and reading it from the payload is just as good. `idIn` is for code that handles events generically, by their tags, without switching on every type.

### An Event About Several Instances

An event is about as many instances of an entity as it is tagged with. A transfer between two accounts carries two `account` tags, and has no single account to answer with:

```java
Tags transfer = Tags.of(ACCOUNT.tag(from), ACCOUNT.tag(to));

ACCOUNT.idsIn(transfer);   // Set.of(from, to)
ACCOUNT.idIn(transfer);    // throws IllegalStateException
```

`idIn` refuses rather than returning whichever tag the hash order put first; the message names both tags and points at `idsIn`. Use `idsIn` wherever an event type can carry more than one id of an entity. Both answer empty when the tags hold no tag of the entity, or only its bare flag.

## The Bare Key: a Flag

`ACCOUNT.tag()` is the key alone, `account`, without a value. It is a flag, for when the presence of the tag is itself the fact.

> Tag matching is exact containment, never a key prefix. A query for the bare `ACCOUNT.tag()` does **not** return events tagged `account:acc-42`. There is no "any account" wildcard: query for the account you mean.
{: .prompt-warning }

## Not Every Tag Is an Entity

A tag whose value is not the identity of anything — a region, a channel, a currency — is an ordinary eventstore tag:

```java
Tags.of(ACCOUNT.tag(accountId), Tag.of("region", "EU"))
```

Declare an entity when the value identifies *one instance of a kind of thing* that events are about and that you will query or lock on. Leave everything else a plain `Tag`. The tag rules — what characters a key and value may hold, why a key cannot contain `:` — are in [Appending Events](/posts/eventstore-appending-events/#what-a-tag-may-contain).

## What Is Stored

Two parts of this are wire format, written into every event, and changing them changes what new events carry and none of the old ones:

| Stored | Where | Consequence of changing it |
|---|---|---|
| **The entity name** | the key of every tag: `account:acc-42` | Renaming `"account"` to `"bankAccount"` means queries for the new key miss every older event |
| **The id record's component name** | inside the payload: `{"accountId":{"value":"acc-42"}, ...}` | Renaming `value` breaks reading every event already written |

What is **not** stored is the id record's class name: the payload serialisation uses no type information, so `AccountId` can be renamed or moved to another package freely. Treat the entity name like a column name — choose it once — and keep the component called `value`.

An entity name may not contain a `:`; `Entity.of("bank:account", ...)` throws, since the eventstore's `Tag` rejects such a key. The name is stripped of surrounding whitespace.

### Two Entities Are Equal When Their Names Are

`Entity` equality is on the name alone, whatever id type each was declared with:

```java
Entity<OtherAccountId> other = Entity.of("account", OtherAccountId::new);
ACCOUNT.equals(other);   // true
```

The name is what an event carries; the id type is a compile-time convenience of the code that declared it. Two declarations of `"account"` in two modules tag events identically, and their tags find each other's events.

## Several Entities in One Event, and the Boundary

Tag an event with every entity it is about. `AccountOpened` is about the account and about the customer, so it carries both — and that is what lets two very different decisions find it:

- "Does account acc-42 exist?" queries `ACCOUNT.tags(accountId)`.
- "How many accounts does customer cust-7 hold?" queries `CUSTOMER.tags(customerId)`.

That an event can belong to several such boundaries at once is the point of DCB, and the reason the type is called `Entity` and not `Aggregate`: an aggregate boundary is a single owner of its events, which is exactly what tags are not. See [Commands and Decision Models](/posts/eventmodeling-commands-and-decision-models/) for how a decision model's query becomes a command's consistency boundary.

A *month*, a *period* or another time-based concept can be an entity too, when events are about it and decisions are made per instance of it. The repository's [`BankingDomainWithClosingTheBooks`](https://github.com/sliceworkz/eventmodeling/blob/develop/sliceworkz-eventmodeling-examples/src/main/java/org/sliceworkz/eventmodeling/examples/banking/BankingDomainWithClosingTheBooks.java) declares a `MONTH` entity with a `MonthId`, and a helper that turns a `YearMonth` into one so every tag spells it the same way.

## The Kernel's Own Tags

Your entity tags are not the only tags on an event. The framework adds tracing tags to everything it appends, all with an `x-` prefix: `x-actor`, `x-channel`, `x-correlation-id`, `x-instance-logical` and friends, and, on events raised in reaction to another event, `x-caused-by`. They say who did what, in which flow, from which process. Matching is containment, so they never get in the way of a query or a boundary built on your entity tags. Do not choose entity names starting with `x-`. [Observability and Tracing](/posts/eventmodeling-observability/) covers what they carry and how to read them.

## Testing with Unpredictable Ids

A command that mints an id with `newId()` raises an event whose id a test cannot know in advance. The published test bases offer a marker for it, `IGNORE_ID()`, which goes through the same factory as every other id:

```java
class OpenAccountCommandTest extends CommandTest<BankingDomainEvent, BankingInboundEvent, BankingOutboundEvent> {

    // domainEventType(), inboundEventType(), outboundEventType() as usual

    CustomerId customer = CUSTOMER.id("cust-1");

    @Test
    void anAccountIsOpenedUnderAFreshId ( ) {
        given()
        .when(new OpenAccountCommand(customer))
        .then()
            .event(new AccountOpened(ACCOUNT.id(IGNORE_ID()), customer, LocalDate.now()),
                   CUSTOMER.tags(customer));
    }
}
```

The test compares the expected and the produced event field by field. Where the expected value is the marker, the produced event must carry the field and its value must be a UUID — only the concrete value is not compared. So `ACCOUNT.id(IGNORE_ID())` against a non-UUID id still fails, and a missing field still fails. For an id that is legitimately not a UUID, `IGNORE_TEXT()` skips the field entirely.

The marker applies to the payload, not to tags: the tag assertion checks that the produced event carries *at least* the expected tags, so leave the minted account's tag out and assert on the ones you know. Fixed ids for seeded history are just `ACCOUNT.id("acc-1")`. [Testing Feature Slices](/posts/eventmodeling-testing/) covers the test bases in full.

## Common Mistakes

| Mistake | What happens | Instead |
|---|---|---|
| `new AccountId(request.get("id"))` | Whitespace survives, so the id differs from the one stored, and queries find nothing | `ACCOUNT.id(request.get("id"))` |
| Validation in the id record's constructor | Every stored event with an id a tightened rule rejects fails to load | Validate in the command, or in `Entity.id(...)` |
| `Tag.of("account", accountId.value())` by hand | Compiles with any string, including a customer id or a misspelt key | `ACCOUNT.tag(accountId)` |
| Querying `ACCOUNT.tag()` for "any account" | Matches only events carrying the bare flag | Query the account you mean |
| `ACCOUNT.idIn(tags)` on a transfer | `IllegalStateException`: two accounts, no single answer | `ACCOUNT.idsIn(tags)` |
| Tagging only with the entity the event is "mainly" about | A decision about the other entity cannot find the event | Tag every entity the event is about |
| Renaming an entity, or the `value` component | New events and old events disagree | Choose both once; rename the record class freely |

## Summary

- Declare an `Entity` per kind of thing your events are about, next to a one-line `EntityId` record: `Entity<AccountId> ACCOUNT = Entity.of("account", AccountId::new)`.
- Mint with `newId()`, normalise with `id(String)`, tag with `tag(id)`/`tags(id)`, read back with `idIn`/`idsIn`.
- One id type per entity is what makes swapped arguments and mistyped tags compile errors.
- Keep the id record lenient, with its component called `value`; put rules in the factory or the command.
- The entity name is the tag key and is stored; so is the component name `value`. The record's class name is not.
- `tag()` is a flag, not a wildcard. Tags that identify nothing are plain `Tag.of(key, value)`.
- In tests, `ACCOUNT.id(IGNORE_ID())` stands for a minted id.

Next: [Commands and Decision Models](/posts/eventmodeling-commands-and-decision-models/).
