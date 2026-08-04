---
layout: post
toc: true
title: Defining Events
description: Defining Domain Events in the Eventstore
date: 2025-11-29 01:00:00
categories: [Eventstore Documentation,Eventstore API]
tags: [events,gdpr,erasable data]
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

## Erasable Data in the event payload

In principle, Event Sourcing states that events are immutable.  In some situations, however, this conflicts with with functional or non-functional (regulatory, ...) requirements where data needs to be deletable from the system.
The European GDRP (General Data Protection Regulation), for example requires the "right to be forgotten", leading to a functional requirement that any personal data must be deleted from the system.

Therefore, the Eventstore separates event data internally in an immutable and erasable (forgettable) part.  Those are combined for everyday usage, but when the erasable part is deleted, the event lives on with only the immutable data.
It will be clear that your application logic will need to be able to process events that have been stripped off of this erasable data.

The EventStore library provides annotations to mark personal data fields that must be erasable for GDPR compliance (right to be forgotten). These annotations enable selective data removal while preserving event structure and non-personal metadata.

### The `@Erasable` Annotation

The `@Erasable` annotation marks fields containing personal data that can be completely erased without losing the event's semantic meaning.

```java
public record CustomerRegistered(
    String customerId,           // Not erasable - needed for correlation

    @Erasable
    String name,                 // Personal data - can be erased

    @Erasable
    String email,                // Personal data - can be erased

    LocalDateTime registeredAt   // Temporal metadata - not erasable
) implements CustomerEvent {}
```

**After erasure:**
```java
// CustomerRegistered(customerId="123", name=null, email=null, registeredAt=2024-01-15T10:30:00)
```

### The `@PartlyErasable` Annotation

The `@PartlyErasable` annotation marks fields containing nested objects that have some (but not all) erasable fields. This signals that erasure logic must recurse into the nested structure.

```java
public record Address(
    @Erasable
    String street,          // Personal data

    @Erasable
    String houseNumber,     // Personal data

    String zipCode,         // Not personal - aggregate data
    String country          // Geographic metadata
) {}

public record CustomerRegistered(
    String customerId,

    @Erasable
    String name,

    @PartlyErasable         // Contains some erasable fields
    Address address
) implements CustomerEvent {}
```

**After erasure:**
```java
// CustomerRegistered(
//   customerId="123",
//   name=null,
//   address=Address(street=null, houseNumber=null, zipCode="12345", country="USA")
// )
```

### When to Use Each Annotation

**Use `@Erasable` when:**
- The entire field value is personal data
- The field can be set to `null` or a sentinel value without breaking semantics
- Examples: name, email, phone number, social security number

**Use `@PartlyErasable` when:**
- The field contains a complex object (record, class, or collection)
- Only some nested fields within the object are personal data
- The structure must be preserved with selective field erasure
- Examples: address objects, contact information blocks, nested value objects

### Multi-Level Nesting

```java
public record ContactInfo(
    @Erasable String email,
    @Erasable String phone
) {}

public record Person(
    @Erasable String name,
    @PartlyErasable ContactInfo contactInfo
) {}

public record CustomerRegistered(
    String customerId,
    @PartlyErasable Person person
) implements CustomerEvent {}
```

### Customizing to your application needs

It could be a good idea to create your own application-level annotation that implies `@Erasable` but adds some documentation fields:

```java
@Target({ElementType.FIELD, ElementType.RECORD_COMPONENT})
@Retention(RetentionPolicy.RUNTIME)
@Erasable
public @interface GdprErasable {
    String purpose() default "";
    Category category() default Category.PERSONAL;
    
    enum Category {
        PERSONAL,      // Name, address, etc.
        CONTACT,       // Email, phone
        FINANCIAL,     // Payment info
        HEALTH,        // Medical data
        BIOMETRIC      // Fingerprints, facial recognition
    }

}
```
You can then use this annotation instead of `@Erasable`, adding the metadata about each field:

```java
public sealed interface CustomerEvent {
	
public record CustomerRegistered (  

	String id,
	
	@GdprErasable(
		category = Category.CONTACT, 
		purpose = "required for personal communication")
	String name,
	
	@GdprErasable(
		category = Category.CONTACT, 
		purpose = "required for sending transactional e-mails")
	String email,
	
	@PartlyErasable
	Address address
	
	) implements CustomerEvent {
	
}

public record Address ( 
	
	@GdprErasable(
		category = Category.PERSONAL, 
		purpose="sending snail mail")
	String street,
	
	@GdprErasable(
		category = Category.PERSONAL, 
		purpose="sending snail mail")
	String number,
	
	String zip ) {
	
}
	
}
```

Generating your obligatory GDPR data register that lists all personal data held, along with the rationale for keeping it, now becomes just a matter of Java reflection on your domain events!

> IMPORTANT: ErasableData can be used to "forget" data in your EventStore, but be aware that PII data will probably still exist in your projections and readmodels.
Make sure to register a domain event that expresses that the right to be forgotten has been used, and remove that data from your readmodels in the projection logic.
{: .prompt-warning }

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
