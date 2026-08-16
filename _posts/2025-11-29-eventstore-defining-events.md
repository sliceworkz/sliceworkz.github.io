---
layout: post
toc: true
title: Defining Events
description: Defining Domain Events in the Eventstore
date: 2025-11-29 01:00:00
categories: [Eventstore Documentation,Eventstore API]
tags: [events,gdpr,crypto-shredding,personal data]
---

This guide covers how to define domain events in your application to use with the Sliceworkz EventStore, As you'll want strongly-typed Events accessible from your event streams. 


## Defining Domain Events

Domain events in the EventStore library are implemented using Java's sealed interfaces combined with record implementations. This approach provides type safety, immutability, and a closed set of possible event types.

### Basic Structure

```java
sealed interface CustomerEvent {
    record CustomerRegistered(String name) implements CustomerEvent {}
    record CustomerNameChanged(String name) implements CustomerEvent {}
    record CustomerChurned() implements CustomerEvent {}
}
```
These records should only contain the application-level information about your event.  Metadata like a unique ID, timestamp, Tags etc... are added as metadata at the time the event is appended to the event log.


### Customizing to your application needs

You can easily extend this basic pattern as per your own requirements, with:
- record structures that are shared over events for data that (eg: an Address record that is both refered to from a CustomerRegistered and CustomerMoved event)
- factory methods to aid in the construction
- builder pattern for more complex events
- interface methods for elements that should be present on all events (eg: customerId(), which would require you to defined a customerId on each and every CustomerEvent implementing record


### Why This Pattern?

**Sealed Interfaces** provide a closed set of domain events. The compiler knows all possible implementations, enabling:
- Exhaustive pattern matching in switch expressions
- Prevention of unauthorized event type extensions
- Clear domain boundaries

**Records** ensure immutability and provide:
- Automatic implementation of constructors, getters, `equals()`, `hashCode()`, and `toString()`
- Concise syntax reducing boilerplate
- Guaranteed immutability (all fields are final)
- Value-based semantics appropriate for events

**Benefits:**
- **Type Safety**: The compiler enforces that only defined event types can be used
- **Immutability**: Events cannot be modified after creation, preserving historical integrity
- **Expressiveness**: Event hierarchies clearly communicate domain concepts
- **Pattern Matching**: Switch expressions on sealed types must handle all cases or fail at compile-time

### Example with Multiple Event Types

```java
sealed interface OrderEvent {
    record OrderPlaced(String orderId, String customerId) implements OrderEvent {}
    record OrderLineAdded(String productId, int quantity) implements OrderEvent {}
    record OrderShipped(String trackingNumber) implements OrderEvent {}
    record OrderCancelled(String reason) implements OrderEvent {}
}
```

### Pattern Matching

Java pattern matching makes event handling code very straightforward:

```java
public void when(CustomerEvent event) {
    switch(event) {
        case CustomerRegistered r -> this.name = r.name();
        case CustomerNameChanged n -> this.name = n.name();
        case CustomerChurned c -> this.active = false;
    }
}
```

## Event Type Names Are Wire Format

**An event class's simple name is stored data.** The name written into storage is `Class.getSimpleName()` — there is no annotation, registry or builder hook to override it. That one string is what goes in the stored event type, what `EventTypesFilter` matches on, and what keys the deserializer when reading events back.

Using the *simple* name rather than the fully qualified one is deliberate, and worth knowing about: moving a class to another package, splitting a hierarchy across packages, or reorganising modules changes nothing on disk. **The package is not a wire commitment. The class name is.**

### Renaming an Event Class Breaks Reads of Its History

Stored events are immutable, so every event already written keeps the old name while the renamed class claims a new one. Reads then fail with:

```
No mapping found for event type 'CustomerRegistered'
```

Every IDE offers that rename as an ordinary refactor, and nothing at compile time objects. Three ways out, in the order you would normally reach for them:

1. **Don't rename.** Pick the stored name deliberately when the event is created, and treat it afterwards the way you would a database column name.
2. **Keep the old name alive in code.** Move a class carrying the old name into a legacy hierarchy, annotate it `@LegacyEvent(upcast = ...)`, and upcast it to the renamed class — see [Approach 2: Upcasting](#approach-2-upcasting) below. This leaves storage untouched and is the only option that needs no access to the database, but it costs a permanent extra class plus an upcaster for what was only a rename.
3. **Rewrite the stored names.** On PostgreSQL this is a valid migration — no foreign key, check constraint or unique index is keyed on the event type:

   ```sql
   UPDATE <prefix>events SET event_type = 'CustomerEnrolled' WHERE event_type = 'CustomerRegistered';
   ```

   On a large table, budget for the row and index rewrite, and scope the statement by `stream_context` when the rename applies to one context only. [`EventStoreImporter`](/posts/eventstore-importing-events/) does the same during a copy, if you would rather rebuild the store than mutate it. Either way, history no longer reads exactly as it was written, and the change has to reach every environment, replica and restored backup — plus anything outside this library reading the same table.

### Names Are Global to a Storage, Not Scoped to a Stream

A stream scopes *reads*; it is not part of a type's identity. Two classes with the same simple name in different contexts write indistinguishable event type values into one table.

- **On one stream this fails loudly.** Registering both throws `IllegalArgumentException: duplicate event name Created`. The message names the string only, not the two classes, so grep for the name to find them.
- **Across streams nothing catches it.** No exception, no warning, at registration or at write time.

**And a read spanning both contexts does not fail cleanly.** A wildcard stream, or a store-wide projection, resolves the payload by name alone. Unknown properties are rejected deliberately, so it looks like a mismatch would be caught — it usually is not. Reading one context's `Created` with the other context's class:

| Reader record vs. stored payload | Outcome |
|---|---|
| more components — `Created(id, amount, dept)` reads `{id, amount}` | **succeeds**, `dept` defaulted to null |
| same component names, different types (`int` → `String`, `int` → `short`) | **succeeds**, coerced |
| same shape, different meaning | **succeeds**, wrong class |
| fewer components — `Created(id)` reads `{id, amount}` | throws |

Only the *narrower* reader is protected. The usual outcome is the wrong class silently populated with another context's data, which surfaces as bad numbers in a projection rather than as an error.

> **Practical rule: keep event class simple names unique across an entire storage, not just per stream.** Two bounded contexts sharing a store cannot both have a `Created`, a `StatusChanged` or an `Updated`. Prefix them (`OrderCreated`, `VacancyCreated`) or give each context its own storage. If two contexts must share a name, keep every read scoped to one stream — no wildcard streams, no store-wide projections — and know that nothing enforces that from here on.
{: .prompt-warning }

## Personal Data in the Event Payload

Event sourcing says events are immutable. GDPR's right to erasure says personal data must be removable on request. The EventStore reconciles those by **crypto-shredding**: personal data is encrypted in place under a key held for the person it belongs to, and erasure destroys that key rather than touching a stored event.

A record component holding personal data is declared `Shreddable<T>` and bound to a `DataSubject`:

```java
public record CustomerRegistered(
    String customerId,                    // pseudonymous — survives erasure
    Shreddable<String> name,              // personal data
    Shreddable<String> email,             // personal data
    LocalDateTime registeredAt            // temporal metadata — not personal
) implements CustomerEvent {}
```

```java
DataSubject alice = DataSubject.of("customer", "alice-42");

stream.append(AppendCriteria.none(), Event.of(
        new CustomerRegistered("alice-42",
                               Shreddable.of("Alice Martin", alice),
                               Shreddable.of("alice@example.org", alice),
                               LocalDateTime.now()),
        Tags.of("customer", "alice-42")));

// later
eventStore.erase(alice, ErasureReason.of("GDPR art.17 request #4711"));

// the event still reads; the personal data does not
event.data().customerId();                    // "alice-42"
event.data().name();                          // Shredded[customer/alice-42/default, k-7f2a91c4]
event.data().name().orElse("[erased]");       // "[erased]"
```

The wrapper, rather than an annotation on a plain field, is what makes this work in a record:

- **A shredded value is never `null`**, so a record whose compact constructor validates its components still builds after its data is gone. Nulling a field instead turns any validating event into a *poison event* that fails every query and every projection over its stream, permanently.
- **"Erased" is distinguishable from "never held any"** — `Shredded` is a state; `null` is not. Nor can a primitive express one, which is why `Shreddable<Integer>` works where an erased `int` would silently read as `0`.
- **A `Shreddable` anywhere works** — nested records, `List` elements, `Map` values — because it is one serializer on one payload document.
- **Two data subjects in one event each get their own key**, so erasing one leaves the other readable.
- **Nothing declared personal can quietly fail to be erasable.** Registering an event type with a `Shreddable` component on a store with no shredding configured fails at `getEventStream`, rather than storing personal data in the clear.

Because personal data is declared in the type system, your obligatory GDPR register of what personal data you hold — and why — is Java reflection over your domain events. Wrap the component in your own annotation to carry the purpose and the retention rule:

```java
public record CustomerRegistered(
    String customerId,

    @PersonalData(purpose = "required for personal communication")
    Shreddable<String> name,

    @PersonalData(purpose = "required for sending transactional e-mails")
    Shreddable<String> email,

    @PersonalData(purpose = "sending physical mail")
    Shreddable<Address> address

) implements CustomerEvent {}
```

> Shredding erases the personal data **in the event log**. It does not reach your read models, caches, search indexes or downstream systems, and projections hold bookmarks so they never re-read the affected events on their own. Register a domain event expressing that the right to be forgotten was exercised, and remove the data from your read models in the projection logic.
{: .prompt-warning }

**See [Erasing Personal Data](/posts/eventstore-erasing-personal-data/)** for data subjects and retention categories, configuring a key store per backend, the audit view, and the contract an implementation must not get wrong.

## Versioning Events

As systems evolve, event structures need to change. The EventStore library supports two primary approaches to event versioning while maintaining event immutability.

### Approach 1: Versioned Event Names

Create new event types with explicit version suffixes:

```java
sealed interface CustomerEvent {
    // Original version
    record CustomerRegistered(String name) implements CustomerEvent {}

    // New version with additional fields
    record CustomerRegisteredV2(Name name, Email email) implements CustomerEvent {}

    record CustomerRenamed(Name name) implements CustomerEvent {}
}
```

**Advantages:**
- Simple and explicit
- Both versions can coexist in the codebase
- Clear distinction between old and new structures

**Disadvantages:**
- Compiler won't stop you from appending new instances of an older event type
- Application code must handle multiple event types for the same business fact
- Queries must explicitly include all versions of an event

### Approach 2: Upcasting

In this approach, you separate between (current) domain events and historical domain events.
The latter still exist in the codebase, but cannot be appended anymore.  They only exists
In addition, each historical event needs to have an Upcaster that transforms it to a current event type.

Since this is all checked at compile-time, this approach is the recommended one.

Define legacy events separately and transform them transparently when reading from the store:

The current ones look just as you would expect them to be, but the naming can give away that things have looked differently in the past:

```java
// Current event definitions
sealed interface CustomerEvent {
    record CustomerRegisteredV2(Name name, Email email) implements CustomerEvent {}
    record CustomerRenamed(Name name) implements CustomerEvent {}
}
```

Historical ones are defined in a parallel sealed interface, annotated as a LegacyEvent with the reference to an Upcaster:

```java
// Historical events (for deserialization only)
sealed interface CustomerHistoricalEvent {
    @LegacyEvent(upcast = CustomerRegisteredUpcaster.class)
    record CustomerRegistered(String name) implements CustomerHistoricalEvent {}

    @LegacyEvent(upcast = CustomerNameChangedUpcaster.class)
    record CustomerNameChanged(String name) implements CustomerHistoricalEvent {}
}

// Upcaster implementation
public class CustomerRegisteredUpcaster
    implements Upcast<CustomerHistoricalEvent.CustomerRegistered,
                      CustomerEvent.CustomerRegisteredV2> {

    @Override
    public List<CustomerEvent.CustomerRegisteredV2> upcast(
            CustomerHistoricalEvent.CustomerRegistered legacy) {
        return List.of(new CustomerEvent.CustomerRegisteredV2(
            new Name(legacy.name()),
            Email.unknown()  // Default for new required field
        ));
    }

    @Override
    public Set<Class<? extends CustomerEvent.CustomerRegisteredV2>> targetTypes() {
        return Set.of(CustomerEvent.CustomerRegisteredV2.class);
    }
}
```

**Usage:**
```java
// Include historical events when creating the stream
EventStream<CustomerEvent> stream = eventstore.getEventStream(
    streamId,
    CustomerEvent.class,              // Current events
    CustomerHistoricalEvent.class     // Historical events
);

// Queries automatically upcast legacy events
stream.query(EventQuery.matchAll())
    .forEach(event -> {
        // All events are of type CustomerEvent
        CustomerEvent current = event.data();
    });
```

**Advantages:**
- Clean separation between current and historical schemas
- Application code only works with current event definitions (compile-time checked)
- Transparent transformation when reading from the store (no additional cognitive load upon application developers)
- Queries can filter on upcasted target types (all historical types are included for free and returned as their corresponding current type)

**Disadvantages:**
- Upcaster implementations needed for each legacy event type
- Slight runtime overhead during event deserialization

### Multi-Event Upcasting

The `Upcast` interface supports three patterns through its `List` return type:

**One-to-one** (most common): A legacy event maps to exactly one current event. Wrap the result in `List.of()`:

```java
@Override
public List<CustomerEvent.CustomerRegisteredV2> upcast(
        CustomerHistoricalEvent.CustomerRegistered legacy) {
    return List.of(new CustomerEvent.CustomerRegisteredV2(
        new Name(legacy.name()),
        Email.unknown()
    ));
}
```

**One-to-many (splitting)**: A legacy event is split into multiple current events. This is useful when a coarse-grained historical event needs to be decomposed into finer-grained events:

```java
public class OrderCreatedUpcaster
    implements Upcast<OrderHistoricalEvent.OrderCreated,
                      OrderEvent> {

    @Override
    public List<OrderEvent> upcast(OrderHistoricalEvent.OrderCreated legacy) {
        return List.of(
            new OrderEvent.OrderPlaced(legacy.orderId(), legacy.customerId()),
            new OrderEvent.OrderLineAdded(legacy.productId(), legacy.quantity())
        );
    }

    @Override
    public Set<Class<? extends OrderEvent>> targetTypes() {
        return Set.of(OrderEvent.OrderPlaced.class, OrderEvent.OrderLineAdded.class);
    }
}
```

When a stored event produces multiple sub-events, each sub-event shares the same `EventReference` id/position/tx but receives a distinct `index` (0, 1, 2, ...) to maintain ordering.

**One-to-zero (filtering)**: An obsolete legacy event can be filtered out entirely by returning an empty list:

```java
public class ObsoleteEventUpcaster
    implements Upcast<LegacyEvent.ObsoleteEvent,
                      CurrentEvent> {

    @Override
    public List<CurrentEvent> upcast(LegacyEvent.ObsoleteEvent legacy) {
        return List.of();  // Event is no longer relevant
    }

    @Override
    public Set<Class<? extends CurrentEvent>> targetTypes() {
        return Set.of();  // No target types
    }
}
```

Filtered events are silently skipped during queries and projection processing. The Projector automatically advances past these "vanished" events.

### Customizing to your application needs

Once you fully understand how event versioning and the Eventstore library works, you could go for more advanced tactics, e.g.:

- When you update the event type column in the eventstore database, and adapt the Java type name directly after, you could rename `CustomerRegisteredV2` back to `CustomerRegistered`, as long as you first rename the original `CustomerRegistered` to e.g. `HistoricalCustomerRegisteredV1`.
This way, all application code keeps the clean current naming, as none of your code will depend on the historical events anyway.  These types are only there to support querying and upcasting old event types from your eventstream.

- If you want (although potentially more controversial) you could even update the event data in the eventstore database and replace it with the upcasted version.
This way, it is as if your old event type never existed.  No more overhead as none of your events will need upcasting, but it violates the idea of immutability and prohibits an older version of the software to read the eventstream up until the point it created it at the time.  
