---
layout: post
toc: true
title: Defining Events
description: Defining Domain Events in the Eventstore
date: 2025-11-29 01:00:00
categories: [Eventstore Documentation,Eventstore API]
tags: [events,gdpr,crypto-shredding,personal data,upcasting,event name]
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

### A Sealed Interface Names Its Whole Hierarchy

A sealed interface is not only a compile-time convenience: in a type filter it stands for every event type under it. `EventTypesFilter.of(CustomerEvent.class)` — or `EventQuery.forTypes(CustomerEvent.class)` — is resolved, when the filter is built, into the records the interface permits, recursively, so the root names the whole hierarchy and a nested sealed interface names its own branch. Events are stored under the simple name of their record, never under an interface, which is why the resolution happens up front: the storage query, the lock check of an append and a projector's filtering then all agree on the same set of stored names.

A non-sealed interface is refused with `IllegalArgumentException`, since nothing says which types it stands for. A filter built from `EventType`s rather than classes is literal: `EventType.of(SomeInterface.class)` names a stored type no record has.

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

**An event class's simple name is stored data, unless the class declares another.** The name written into storage is `EventType.of(Class)`: `Class.getSimpleName()`, or the value of an `@EventName` annotation on the class. That one string is what goes in the stored event type, what `EventTypesFilter` matches on, and what keys the deserializer when reading events back.

Using the *simple* name rather than the fully qualified one is deliberate, and worth knowing about: moving a class to another package, splitting a hierarchy across packages, or reorganising modules changes nothing on disk. **The package is not a wire commitment. The class name is** — for every class that does not carry `@EventName`, which should be most of them.

`EventType` has two factories: `EventType.of(SomeEvent.class)` and `EventType.named("SomeEvent")`. There is deliberately no factory taking an event *instance*: an overload on `Object` would accept a stored name passed by mistake and quietly produce the type named `String`. Code holding an instance writes `EventType.of(event.getClass())`.

### `@EventName`: a Stored Name That Is Not the Class Name

```java
@EventName("CustomerRegistered")
record CustomerSignedUp ( String id, String name ) implements CustomerEvent { }
```

The annotation's value is used exactly as given — non-blank, with no leading or trailing whitespace, or `getEventStream(...)` throws `IllegalArgumentException` — on append, in a stream's type mappings, in `EventTypesFilter.of(Class...)` and on a `@LegacyEvent`. It is not inherited, and it combines with `@LegacyEvent`.

**The plain class name is the intended setup.** `@EventName` is for a class whose stored name *cannot* be its own name: a renamed class, or a simple name another context already stores. Annotating every event up front buys nothing — a string literal is as permanent a commitment as a class name, so it does not avoid the rename problem, it only adds a second name to keep in step with the first.

### Renaming an Event Class Breaks Reads of Its History

Stored events are immutable, so every event already written keeps the old name while the renamed class claims a new one. Reads then fail with:

```
No mapping found for event type 'CustomerRegistered'
```

Every IDE offers that rename as an ordinary refactor, and nothing at compile time objects. Four ways out, in the order you would normally reach for them:

1. **Don't rename.** Pick the stored name deliberately when the event is created, and treat it afterwards the way you would a database column name. This is the intended setup and needs no annotation.
2. **Rename the class, keep the stored name.** Annotate the renamed class with the name its history was written under — `@EventName("CustomerRegistered")` on `CustomerSignedUp`, as above. Storage is untouched, nothing is upcast and no database access is needed. The cost is that the class and its stored name now differ, permanently, which the annotation makes visible at the declaration. This fixes a rename; when the *shape* changed too, it is not enough on its own.
3. **Keep the old name alive in code.** Move a class carrying the old name into a legacy hierarchy, annotate it `@LegacyEvent(upcaster = ...)`, and upcast it to the renamed class — see [Approach 2: Upcasting](#approach-2-upcasting) below. The legacy class can carry the stored name in an `@EventName` too, so it need not be called what the history is called. This is the option when the event's shape changed as well as its name; for a bare rename it costs a permanent extra class plus an upcaster that option 2 does not.
4. **Rewrite the stored names.** On PostgreSQL this is a valid migration — no foreign key, check constraint or unique index is keyed on the event type:

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

> **Practical rule: keep stored event names unique across an entire storage, not just per stream.** Two bounded contexts sharing a store cannot both *store* a `Created`, a `StatusChanged` or an `Updated`. They can both have a *class* called that: give one of them a distinct stored name with `@EventName("OrderCreated")`, prefix the class names themselves (`OrderCreated`, `VacancyCreated`), or give each context its own storage. With the stored names distinct, both hierarchies register on one wildcard stream and a store-wide read resolves each context's events with its own class. If two contexts must share a stored name, keep every read scoped to one stream — no wildcard streams, no store-wide projections — and know that nothing enforces that from here on.
{: .prompt-warning }

## Personal Data in the Event Payload

Event sourcing says events are immutable. GDPR's right to erasure says personal data must be removable on request. The EventStore reconciles those by **crypto-shredding**: personal data is encrypted in place under a key held for the person it belongs to, and erasure destroys that key rather than touching a stored event.

A record component holding personal data is declared `Shreddable<T>` and bound to a `DataSubject`:

```java
public record CustomerRegistered(
    String customerId,                    // pseudonymous — survives erasure
    Shreddable<String> name,              // personal data
    Shreddable<String> email,             // personal data
    Instant registeredAt                  // temporal metadata — not personal
) implements CustomerEvent {}
```

```java
DataSubject alice = DataSubject.of("customer", "alice-42");

stream.append(Event.of(
        new CustomerRegistered("alice-42",
                               Shreddable.of("Alice Martin", alice),
                               Shreddable.of("alice@example.org", alice),
                               Instant.now()),
        Tags.of("customer", "alice-42")));

// later: erase the person, across every category their data was written under
eventStore.erase("customer", "alice-42", ErasureReason.of("GDPR art.17 request #4711"));

// the event still reads; the personal data does not
event.data().customerId();                    // "alice-42"
event.data().name();                          // Shredded[customer/alice-42/default, k-7f2a91c4]
event.data().name().orElse("[erased]");       // "[erased]"
```

The wrapper, rather than an annotation on a plain field, is what makes this work in a record:

- **A shredded value is never `null`**, so a record whose compact constructor validates its components still builds after its data is gone. Nulling a field instead turns any validating event into a *poison event* that fails every query and every projection over its stream, permanently.
- **"Erased" is distinguishable from "never held any"** — `Shredded` is a state; `null` is not. So is `Withheld`, the answer a reader gets for data it is not entitled to read. Nor can a primitive express one, which is why `Shreddable<Integer>` works where an erased `int` would silently read as `0`.
- **A `Shreddable` anywhere works** — nested records, `List` elements, `Map` values — because it is one serializer on one payload document.
- **Two data subjects in one event each get their own key**, so erasing one leaves the other readable.
- **Nothing declared personal can quietly fail to be erasable.** Registering an event type with a `Shreddable` component on a store with no shredding configured fails at `getEventStream`, rather than storing personal data in the clear. A `Shreddable` hidden where that check cannot see it — behind a component declared as an interface — fails the append instead, as an `EventSerializationException` with nothing stored.

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

**See [Erasing Personal Data](/posts/eventstore-erasing-personal-data/)** for data subjects and retention categories, configuring a key store per backend, readers that may not see personal data, the audit view, and the contract an implementation must not get wrong.

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

In this approach, you separate between (current) domain events and legacy domain events.
The latter still exist in the codebase, but cannot be appended anymore: they only exist to read history.
In addition, each legacy event needs an `Upcaster` that transforms it into a current event type — or into the next legacy version, which is upcast in turn.

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

Legacy ones are defined in a parallel sealed interface, annotated as a `@LegacyEvent` naming its upcaster:

```java
// Legacy events (for deserialization only)
sealed interface CustomerLegacyEvent {
    @LegacyEvent(upcaster = CustomerRegisteredUpcaster.class)
    record CustomerRegistered(String name) implements CustomerLegacyEvent {}

    @LegacyEvent(upcaster = CustomerNameChangedUpcaster.class)
    record CustomerNameChanged(String name) implements CustomerLegacyEvent {}
}

// Upcaster implementation: one target type, so targetTypes() is derived from the type argument
public class CustomerRegisteredUpcaster
    implements Upcaster<CustomerLegacyEvent.CustomerRegistered,
                        CustomerEvent.CustomerRegisteredV2> {

    @Override
    public List<CustomerEvent.CustomerRegisteredV2> upcast(
            CustomerLegacyEvent.CustomerRegistered legacy) {
        return List.of(new CustomerEvent.CustomerRegisteredV2(
            new Name(legacy.name()),
            Email.unknown()  // Default for new required field
        ));
    }
}
```

An upcaster needs a public no-argument constructor. **`targetTypes()` defaults to the `TARGET_EVENT` type argument** when the declaration fixes one event class, as here — so the ordinary one-to-one upcaster is the `upcast` method alone. Where the declaration does not fix one class (a sealed interface as the argument, which is how an upcaster that splits or drops its event is typed), the default throws, and `getEventStream` reports it naming the upcaster and the fix: override `targetTypes()`.

**Usage:**
```java
// Include legacy events when creating the stream
EventStream<CustomerEvent> stream = eventstore.getEventStream(
    streamId,
    CustomerEvent.class,          // Current events
    CustomerLegacyEvent.class     // Legacy events
);

// Queries automatically upcast legacy events
for (Event<CustomerEvent> event : stream.query(EventQuery.matchAll())) {
    // All events are of type CustomerEvent
    CustomerEvent current = event.data();
}
```

**Advantages:**
- Clean separation between current and historical schemas
- Application code only works with current event definitions (compile-time checked)
- Transparent transformation when reading from the store (no additional cognitive load upon application developers)
- Queries can filter on upcasted target types (all historical types are included for free and returned as their corresponding current type)

**Disadvantages:**
- Upcaster implementations needed for each legacy event type
- Slight runtime overhead during event deserialization

### What Is Checked, and When

- **At stream creation**, every class `targetTypes()` names must be a type registered on the stream — a current type, or a further legacy type — and every chain must end in a current type. A target the stream does not register (a class from a hierarchy you forgot to pass to `getEventStream`) and a cycle are both `IllegalArgumentException`, naming the upcaster and the target. So are a `@LegacyEvent` on a class registered as current, a current class registered as legacy, and an upcaster that cannot be instantiated.
- **On the read**, an event produced by an upcaster whose class is not among its declared targets fails as an `EventDeserializationException` naming the upcaster, the class produced and the declared set. An upcaster declaring `Set.of()` and producing an event is the shape this catches: a query for the produced type would never have fetched the event it came from.
- **A filter names current types.** A query or an `AppendCriteria` whose type filter names a *legacy* type is refused with `IllegalArgumentException`, naming the current type it is read as. Storage would fetch the legacy rows, upcast them into the current type, and then drop every one for not matching the legacy filter — a query returning nothing, while the same filter as a consistency boundary would count those very events. Name the current type: a query over it fetches its legacy events too.

### Upcast Chains

An upcaster's target may itself be a `@LegacyEvent`, and the chain is followed until it reaches a current type. A history written as `V1`, then `V2`, then `V3` reads through a `V1 → V2` upcaster and a `V2 → V3` one, each written when its version arrived and neither rewritten when the next came:

```java
sealed interface CustomerLegacyEvent {
    @LegacyEvent(upcaster = V1ToV2.class)
    record CustomerRegistered(String name) implements CustomerLegacyEvent {}

    @LegacyEvent(upcaster = V2ToV3.class)
    record CustomerRegisteredV2(String name, String email) implements CustomerLegacyEvent {}
}

// the target is a legacy type: V2ToV3 runs next
public class V1ToV2 implements Upcaster<CustomerLegacyEvent.CustomerRegistered, CustomerLegacyEvent.CustomerRegisteredV2> {
    @Override
    public List<CustomerLegacyEvent.CustomerRegisteredV2> upcast(CustomerLegacyEvent.CustomerRegistered v1) {
        return List.of(new CustomerLegacyEvent.CustomerRegisteredV2(v1.name(), null));
    }
}

public class V2ToV3 implements Upcaster<CustomerLegacyEvent.CustomerRegisteredV2, CustomerEvent.CustomerRegisteredV3> {
    @Override
    public List<CustomerEvent.CustomerRegisteredV3> upcast(CustomerLegacyEvent.CustomerRegisteredV2 v2) {
        return List.of(new CustomerEvent.CustomerRegisteredV3(new Name(v2.name()), Email.ofNullable(v2.email())));
    }
}
```

A stored `CustomerRegistered` then reads as a `CustomerRegisteredV3`, and a query or a consistency boundary over `CustomerRegisteredV3` fetches the stored `CustomerRegistered` and `CustomerRegisteredV2` events too. When a later hop fails, the `EventDeserializationException` names the *stored* type — the event you would dead-letter — and the message names the upcaster of the hop that threw, which is the code to fix.

### Multi-Event Upcasting

The `Upcaster` interface supports three patterns through its `List` return type:

**One-to-one** (most common): A legacy event maps to exactly one current event. Wrap the result in `List.of()`:

```java
@Override
public List<CustomerEvent.CustomerRegisteredV2> upcast(
        CustomerLegacyEvent.CustomerRegistered legacy) {
    return List.of(new CustomerEvent.CustomerRegisteredV2(
        new Name(legacy.name()),
        Email.unknown()
    ));
}
```

**One-to-many (splitting)**: A legacy event is split into multiple current events. This is useful when a coarse-grained historical event needs to be decomposed into finer-grained events:

```java
public class OrderCreatedUpcaster
    implements Upcaster<OrderLegacyEvent.OrderCreated,
                        OrderEvent> {

    @Override
    public List<OrderEvent> upcast(OrderLegacyEvent.OrderCreated legacy) {
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

The type argument here is the sealed `OrderEvent`, so the default `targetTypes()` cannot tell which records the upcaster produces and throws — which is why this one declares them. Declaring the whole hierarchy would be correct and silently wasteful: every query for any `OrderEvent` type would then fetch these legacy events only to discard what they upcast into.

When a stored event produces multiple sub-events, each sub-event shares the same `EventReference` id/position/tx but receives a distinct `index` (0, 1, 2, ...) to maintain ordering.

**One-to-zero (filtering)**: An obsolete legacy event can be filtered out entirely by returning an empty list:

```java
public class ObsoleteEventUpcaster
    implements Upcaster<CustomerLegacyEvent.ObsoleteEvent,
                        CustomerEvent> {

    @Override
    public List<CustomerEvent> upcast(CustomerLegacyEvent.ObsoleteEvent legacy) {
        return List.of();  // Event is no longer relevant
    }

    @Override
    public Set<Class<? extends CustomerEvent>> targetTypes() {
        return Set.of();  // No target types
    }
}
```

Filtered events are silently skipped during queries and projection processing. The Projector automatically advances past these "vanished" events. `getEventById` still tells such an event apart from one that does not exist: it answers a *present* result holding an empty list — see [Querying by Event ID](/posts/eventstore-querying-events/#querying-by-event-id).

### Customizing to your application needs

Once you fully understand how event versioning and the Eventstore library works, you could go for more advanced tactics, e.g.:

- You can give the current event back its clean name without touching the database: rename `CustomerRegisteredV2` to `CustomerRegistered` and the legacy class to e.g. `CustomerRegisteredV1`, then put `@EventName("CustomerRegistered")` on the legacy class and `@EventName("CustomerRegisteredV2")` on the current one, so both keep the stored names their history was written under.
This way, all application code keeps the clean current naming, as none of your code will depend on the legacy events anyway.  These types are only there to support querying and upcasting old event types from your eventstream.

- If you want (although potentially more controversial) you could even update the event data in the eventstore database and replace it with the upcasted version.
This way, it is as if your old event type never existed.  No more overhead as none of your events will need upcasting, but it violates the idea of immutability and prohibits an older version of the software to read the eventstream up until the point it created it at the time.  
