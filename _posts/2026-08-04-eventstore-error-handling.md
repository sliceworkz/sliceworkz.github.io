---
layout: post
toc: true
title: Error Handling
description: The exceptions the EventStore throws, and which of them are worth retrying
date: 2026-08-04 04:00:00
categories: [Eventstore Documentation,Eventstore API]
tags: [exceptions,error handling,retry,serialization,poison event]
---

This guide covers the exceptions the EventStore throws, where each comes from, and — the question that actually decides your code — which of them a retry can clear.

## The Exceptions, and What to Do About Each

Everything is unchecked. Nothing forces a `catch` you did not want.

| Exception | Package | Thrown by | Retry? |
|---|---|---|---|
| `OptimisticLockingException` | `…eventstore.stream` | a conditional `append` whose consistency boundary moved | **Yes, immediately** — re-read, re-decide, re-append |
| `EventStorageException` | `…eventstore.spi` | the storage: connection lost, schema invalid, result limit exceeded | **Maybe, with backoff** — depends on the cause |
| `EventStorageClosedException` | `…eventstore.spi` | any operation on a closed storage or store | **No** — the object is gone for good |
| `EventSerializationException` | `…eventstore.events` | `append`, for a payload that cannot be written | **Never** |
| `EventDeserializationException` | `…eventstore.events` | reading a stored event this stream's mappings cannot read | **Never** |
| `EventImportConflictException` | `…eventstore.spi` | `importEvents` hitting an existing id or idempotency key | **No** — change the mode or the data |
| `ProjectorException` | `…eventstore.projection` | a `Projector` run — wraps whatever it caught | inspect `getCause()` |
| `IllegalArgumentException` | — | a misconfigured `getEventStream(...)` call, an illegal `Tag`, an incompatible query combination | **No** — a code fix |

**There is deliberately no common root** for "anything this library throws". A root only pays for itself if catching that is a useful operation, and it is not: these failures need opposite responses. A boundary conflict wants an immediate retry, a dropped connection wants backoff, an unreadable payload wants neither. A root would mostly encourage the broad catch that this split exists to avoid.

## Optimistic Locking: The Expected Failure

`OptimisticLockingException` is not an error condition. It is the DCB mechanism working: a new relevant fact emerged between the moment a decision was taken and the moment its result was appended.

```java
EventQuery relevant = EventQuery.forEvents(EventTypesFilter.any(), Tags.of("customer", "123"));

for ( int attempt = 0; attempt < 5; attempt++ ) {
    List<Event<CustomerEvent>> facts = stream.query(relevant).toList();

    if ( !decide(facts) ) {
        return;                                  // the new facts changed the decision
    }

    try {
        stream.append(
            AppendCriteria.of(relevant, facts.isEmpty() ? null : facts.getLast().reference()),
            Event.of(new CustomerNameChanged("123", "Jane"), Tags.of("customer", "123")));
        return;
    } catch ( OptimisticLockingException e ) {
        // somebody appended a relevant fact — read again and decide again
    }
}
throw new IllegalStateException("gave up after 5 contended attempts");
```

Two things matter about the loop. **Re-read and re-decide**, do not merely re-append: appending the same conclusion against a fresh reference defeats the point of the check. And **bound the attempts**, because a boundary that is genuinely hot will not clear on its own.

The exception carries the boundary that fired, which is worth logging when contention is unexpected:

```java
catch ( OptimisticLockingException e ) {
    LOGGER.debug("boundary {} moved past {}", e.getFilter(), e.getExpectedLastEventReference());
}
```

> An **empty** `getExpectedLastEventReference()` under a real filter is not "no criteria". It means the decision was taken on an empty result — which is still a consistency boundary, so any matching event in the stream is a new relevant fact and correctly raises. Only `AppendCriteria.none()` skips the check entirely.
{: .prompt-info }

Note that this exception is also the mechanism behind [DCB-style idempotency](/posts/eventstore-appending-events/#dcb-style-idempotency), where catching and ignoring it is exactly right — there it signals successful deduplication, not contention.

## Storage Failures

`EventStorageException` means the storage could not do what was asked. Its realistic causes span a wide range, and the cause is what tells them apart:

- a dropped or refused connection — transient, retry with backoff
- schema validation failing at startup — fatal, and better fixed than retried
- a query exceeding the configured `resultLimit` — a query-design problem, not a transient one
- the LISTEN/NOTIFY channels failing to establish within the startup deadline — see [Lifecycle and Shutdown](/posts/eventstore-lifecycle/)
- `placeBookmark` with a reference this store never stored — a caller error that a retry cannot clear; nothing is written and the reader's previous bookmark stands. See [Bookmarking](/posts/eventstore-bookmarking/#the-reference-must-name-a-stored-event)

`EventStorageClosedException` is a subclass, and is the one case with no ambiguity: every read and write on a closed storage or store throws it, permanently. There is no reopening. If you see it, something closed a store that is still in use — usually a shutdown hook running while requests are still in flight.

```java
catch ( EventStorageClosedException e ) {
    // do not retry: this store is gone. fail the request.
} catch ( EventStorageException e ) {
    // possibly transient: retry with backoff, or fail after N attempts
}
```

`name()` keeps working on a closed storage, so log lines that identify the store do not themselves start throwing.

## Payload Failures: Serialization and Deserialization

Two named types cover payload conversion. Both live in the **api** module, in `org.sliceworkz.eventstore.events`, so catching one never means importing from an implementation package.

**Neither is ever worth retrying, and that is the whole point of the split.** A failure to convert a payload is a property of the payload and the type mappings: identical on the next attempt and on every other instance. An `EventStorageException` from the same call may be a dropped connection. A retry loop that cannot tell them apart either retries forever on an unreadable event or gives up on a blip.

### EventSerializationException

Thrown from `append`, for a payload that cannot be written. **Nothing is stored.** It carries `getEventType()`.

In practice this means a domain event carrying something Jackson cannot write — a non-serializable field, a cyclic reference, a type with no accessible components.

### EventDeserializationException

Thrown when a stored event cannot be read with this stream's type mappings. It carries `getEventType()` — the name in *storage*, which is not necessarily a type any current class claims — and `getReference()`.

**This is a poison event, not a broken store.** The storage read succeeded. The realistic causes are configuration and history rather than bugs:

- a stream opened without a root class covering a stored type
- a record that has since lost a component the stored JSON still carries (unknown properties fail deliberately)
- a renamed event class — see [event type names are wire format](/posts/eventstore-defining-events/#event-type-names-are-wire-format)
- an `@Upcast` method throwing on legacy data that does not satisfy a current validation rule

`getReference()` is what makes the type useful rather than merely tidy. The serde layer is handed a type name and two JSON strings and cannot say *which* stored event failed, so the reference is attached on the way out. Its `id()` goes to `getEventById` on a **raw** stream — one with no mappings has nothing to fail on — so the stored JSON can be read even though the typed stream chokes on it:

```java
catch ( EventDeserializationException e ) {
    LOGGER.error("cannot read stored event of type {} at {}", e.getEventType(), e.getReference());

    e.getReference().ifPresent(ref -> {
        EventStream<Object> raw = eventStore.getEventStream(EventStreamId.anyContext());
        raw.getEventById(ref.id()).forEach(stored -> LOGGER.error("raw payload: {}", stored.data()));
    });
}
```

### Deserialization Is Lazy

`query()` does not deserialize as it returns — it surfaces from **your** terminal operation on the stream:

```java
Stream<Event<CustomerEvent>> events = stream.query(EventQuery.matchAll());   // no throw here
List<Event<CustomerEvent>> list = events.toList();                          // throws here
```

Two calls behave differently. `getEventById` is eager and throws directly. And `append` deserializes the events it just wrote in order to return them — so a payload that serializes but cannot be read back fails *there*, as a deserialization failure, with the event already stored.

## Misconfiguration Is IllegalArgumentException

Three registration-time mistakes are properties of the `Class` handed to `getEventStream(...)`, and fail before anything is read or written:

- `@LegacyEvent` on a class registered as a **current** type
- a current class registered as a **legacy** type
- an upcaster that cannot be instantiated

They join two checks that were always typed this way — a duplicate event name, and a non-sealed interface. There is no recovery but to fix the code, so they are not serde failures and no retry applies. The messages name the upcaster *and* the event class and keep the reflective cause.

`Tag.of(...)` and `Tags.of(...)` also reject shapes that would not survive storage — see [tag construction rules](/posts/eventstore-appending-events/#what-a-tag-may-contain).

Query combination has its own rules, and violating them throws here too: `combineWith` rejects differing `until` references, differing directions, and any query carrying a limit.

## Failures Inside a Projector

`Projector.run()` wraps everything it catches in a `ProjectorException`, so a dropped connection and an unreadable event arrive identically. **The type of `getCause()` is the only signal a caller has:**

```java
try {
    projector.run();
} catch ( ProjectorException e ) {
    switch ( e.getCause() ) {
        case EventDeserializationException poison ->
            // never retry: quarantine the event, or fix the mappings
            quarantine(poison.getReference().orElse(null));
        case EventStorageException storage ->
            // possibly transient: schedule another run
            scheduleRetry();
        default ->
            // the projection's own handler threw
            LOGGER.error("projection failed", e.getCause());
    }
}
```

> `ProjectorException.getEventReference()` is the last event **handled**, and the offending event never reached the projection. When the cause is a deserialization failure, `EventDeserializationException.getReference()` is the one that names the culprit.
{: .prompt-warning }

A `BatchAwareProjection` whose `afterBatch` throws — a commit that failed — arrives the same way: a `ProjectorException` carrying that failure as its cause. The batch did not land, so the projector's cursor goes back to where the batch started and those events are offered again on the next run. If the rollback in `cancelBatch` *also* threw, that second failure is attached as a **suppressed** exception rather than replacing the cause, so `getCause()` still names what actually went wrong. See [Batch-Aware Projections](/posts/eventstore-projecting-events/#what-the-batch-boundary-guarantees).

## Failures Inside a Listener

A listener's exception is **contained**, logged at ERROR, and the next subscriber still gets the notification. It is never anybody else's failure, and never silent.

Three consequences worth designing around:

- **The appending caller is not told.** `append()` has already committed by the time listeners run, so a listener cannot fail an append, be rolled back with it, or veto it.
- **Nothing replays what a failing listener missed.** It is notified again on the next append; the notification it failed on is gone.
- **A listener that must not lose progress belongs behind a `Projector` reading from a bookmark**, which resumes from a persisted position rather than from whatever the last notification happened to be.

See [Eventstore Listeners](/posts/eventstore-listeners/) for the full delivery contract.

## A Note on Catching Broadly

Because everything is unchecked and there is no shared root, `catch (RuntimeException e)` around an event store call catches all of it — including the boundary conflict you meant to retry and the poison event you meant to quarantine. Where a call site has a policy at all, catch by name:

```java
try {
    stream.append(criteria, event);
} catch ( OptimisticLockingException e ) {
    // retry: re-read, re-decide
} catch ( EventSerializationException e ) {
    // a code bug: nothing was stored, no retry will help
} catch ( EventStorageException e ) {
    // infrastructure: retry with backoff, then fail
}
```

Where it has no policy, let it propagate. An exception that reaches a request boundary and produces a 500 with a stack trace is more useful than one absorbed into a log line.
