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
| `IdempotencyKeyConflictException` | `…eventstore.stream` | an `append` whose batch mixes idempotency keys already stored with new ones | **Never** — the keys are derived wrongly |
| `EventStorageException` | `…eventstore.spi` | the storage: connection lost, schema invalid, result limit exceeded | **Maybe, with backoff** — depends on the cause |
| `EventStorageClosedException` | `…eventstore.spi` | any operation on a closed storage or store | **No** — the object is gone for good |
| `EventSerializationException` | `…eventstore.events` | `append`, for a payload that cannot be written | **Never** |
| `EventDeserializationException` | `…eventstore.events` | reading a stored event this stream's mappings cannot read | **Never** |
| `EventImportConflictException` | `…eventstore.spi` | `importEvents` hitting an existing id or idempotency key | **No** — change the mode or the data |
| `ShreddingException` | `…eventstore.shredding` | the key store protecting personal data cannot be reached, holds no such key, or an envelope is malformed | **Maybe, with backoff** — an outage clears; a miswired key store or a malformed envelope does not |
| `ProjectorException` | `…eventstore.projection` | a `Projector` run — wraps whatever it caught | inspect `getCause()` |
| `IllegalArgumentException` | — | a misconfigured `getEventStream(...)` call, an illegal `Tag`, an incompatible `or`, a filter naming a legacy type, an append through a wildcard stream, a batch repeating an idempotency key | **No** — a code fix |

**There is deliberately no common root** for "anything this library throws". A root only pays for itself if catching that is a useful operation, and it is not: these failures need opposite responses. A boundary conflict wants an immediate retry, a dropped connection wants backoff, an unreadable payload wants neither. A root would mostly encourage the broad catch that this split exists to avoid.

## Optimistic Locking: The Expected Failure

`OptimisticLockingException` is not an error condition. It is the DCB mechanism working: a new relevant fact emerged between the moment a decision was taken and the moment its result was appended.

```java
EventQuery relevant = EventQuery.forTags(Tags.of("customer", "123"));

for ( int attempt = 0; attempt < 5; attempt++ ) {
    EventReference head = stream.head().orElse(null);          // pin the boundary first
    List<Event<CustomerEvent>> facts = stream.query(relevant.until(head));

    if ( !decide(facts) ) {
        return;                                  // the new facts changed the decision
    }

    try {
        stream.append(
            AppendCriteria.of(relevant, head),   // the query WITHOUT its until
            Event.of(new CustomerNameChanged("123", "Jane"), Tags.of("customer", "123")));
        return;
    } catch ( OptimisticLockingException e ) {
        // somebody appended a relevant fact — take the head again, read again, decide again
    }
}
throw new IllegalStateException("gave up after 5 contended attempts");
```

Three things matter about the loop. **Re-read and re-decide**, do not merely re-append: appending the same conclusion against a fresh reference defeats the point of the check. **Bound the attempts**, because a boundary that is genuinely hot will not clear on its own — measured, a boundary shared by sixteen writers conflicts 82% of the time, and adding writers makes useful throughput strictly worse; widen the boundary instead. And **never hand the criteria the bounded query**: `AppendCriteria.of(relevant.until(head), head)` never raises this exception at all, because nothing after the head is relevant to it — the loop then "succeeds" every time.

The exception carries the boundary that fired, which is worth logging when contention is unexpected:

```java
catch ( OptimisticLockingException e ) {
    LOGGER.debug("boundary {} moved past {}", e.getFilter(), e.getExpectedLastEventReference());
}
```

The exception names the boundary *you* decided on, even where the store traced your current types back to the legacy types that upcast into them to check it.

> An **empty** `getExpectedLastEventReference()` under a real filter is not "no criteria". It means the decision was taken on an empty result — which is still a consistency boundary, so any matching event in the stream is a new relevant fact and correctly raises. Only `AppendCriteria.none()` skips the check entirely.
{: .prompt-info }

Note that this exception is also the mechanism behind [DCB-style idempotency](/posts/eventstore-appending-events/#dcb-style-idempotency), where catching and ignoring it is exactly right — there it signals successful deduplication, not contention.

## Idempotency Key Conflicts

`IdempotencyKeyConflictException` is thrown by an append whose batch mixes idempotency keys that are already stored on the stream with keys that are not. A genuine retry finds every key stored, and is swallowed whole; a batch that finds only some cannot be a retry — one of its events collides with a *different* event holding its key — so it is refused, with nothing stored. It extends `RuntimeException` directly rather than `EventStorageException`, because a retry fails identically: the fix is in how the command derives its keys. See [Idempotency](/posts/eventstore-appending-events/#a-batch-is-the-unit-of-de-duplication).

## Storage Failures

`EventStorageException` means the storage could not do what was asked. Its realistic causes span a wide range, and the cause is what tells them apart:

- a dropped or refused connection — transient, retry with backoff
- schema validation failing at startup — fatal, and better fixed than retried
- a query exceeding the configured `resultLimit` — a query-design problem, not a transient one
- a conditional append that waited longer than `lockTimeout` for its stream's lock (SQLSTATE `55P03` on the cause) — a stalled lock holder; retrying later is reasonable, retrying in a tight loop is not. See [Timeouts](/posts/eventstore-configuring-postgresql-storage/#timeouts-a-stalled-lock-holder-and-a-socket-that-dies-silently)
- the LISTEN/NOTIFY channels failing to establish within the startup deadline — see [Lifecycle and Shutdown](/posts/eventstore-lifecycle/)
- a store restored logically into a younger PostgreSQL cluster, refused at startup — fatal; see [Backup and Restore](/posts/eventstore-configuring-postgresql-storage/#backup-and-restore)
- no `db.properties` at any of the locations the builder looks — the message names every location it tried
- `placeBookmark` with a reference this store never stored — a caller error that a retry cannot clear; nothing is written and the reader's previous bookmark stands. See [Bookmarking](/posts/eventstore-bookmarking/#the-reference-must-name-a-stored-event)
- an `importEvents` or raw SPI append whose payload is not a JSON document — nothing of the batch is stored, on every backend

`EventStorageClosedException` is a subclass, and is the one case with no ambiguity: every read and write on a closed storage or store throws it, permanently. There is no reopening. If you see it, something closed a store that is still in use — usually a shutdown hook running while requests are still in flight.

```java
catch ( EventStorageClosedException e ) {
    // do not retry: this store is gone. fail the request.
} catch ( EventStorageException e ) {
    // possibly transient: retry with backoff, or fail after N attempts
}
```

`name()` keeps working on a closed storage, so log lines that identify the store do not themselves start throwing.

## Key Store Failures

`ShreddingException` means the key store behind [crypto-shredding](/posts/eventstore-erasing-personal-data/) could not answer: an unreachable Vault, an expired token, a timeout, a corrupt sealed envelope, an algorithm the codec does not implement.

**It is never how an erased value is reported, nor a withheld one.** A key that has been destroyed is not a failure — it is the mechanism working, and it surfaces as `Shreddable.Shredded` on the event, with the read succeeding. A reader that is not entitled to a value gets `Shreddable.Withheld`, and its read succeeds too. `ShreddingException` is for everything else — including a key id this store has **never held**, which means the key store is not the one the events were sealed against: the wrong directory, the wrong prefix, events imported without their keys. Reported as erased, that would silently erase the whole store at once.

That distinction decides your retry, and it is the reason this exception exists as its own type rather than being folded into deserialization. A key-store outage is transient: the same read succeeds once the store is reachable again. The typed serde therefore rethrows a `ShreddingException` **unwrapped** rather than letting it arrive as an `EventDeserializationException`, which means *never retry*.

```java
catch ( ShreddingException e ) {
    // the key store is unhappy — back off and read again; do NOT treat this as erased data
}
```

The stakes are highest on the projection side. Projections are at-least-once and advance a bookmark past what they have handled, so an outage misreported as erasure would be written into read models permanently and never revisited. Reported as an exception, the read fails loudly, the bookmark does not move, and the projector recovers on its next run.

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
- an `Upcaster` throwing on legacy data that does not satisfy a current validation rule — reported as an upcaster failure, naming the upcaster of the hop that threw, not as a parse failure
- an upcaster producing an event outside the types its `targetTypes()` declares

`getReference()` is what makes the type useful rather than merely tidy. The serde layer is handed a type name and two JSON strings and cannot say *which* stored event failed, so the reference is attached on the way out. Its `id()` goes to `getEventById` on a **raw** stream — one with no mappings has nothing to fail on — so the stored JSON can be read even though the typed stream chokes on it:

```java
catch ( EventDeserializationException e ) {
    LOGGER.error("cannot read stored event of type {} at {}", e.getEventType(), e.getReference());

    e.getReference().ifPresent(ref -> {
        EventSource<String> raw = eventStore.getRawEventStream(EventStreamId.anyContext());
        raw.getEventById(ref.id()).ifPresent(stored ->
            LOGGER.error("raw payload: {}", stored.getFirst().data()));   // the stored JSON document
    });
}
```

### It Is Thrown by the Read Itself

`query()` and `page()` read their result in full before returning it, so a poison event fails **the call that reads it**, and nothing is returned — a query or page holding one hands out none of its events:

```java
List<Event<CustomerEvent>> events = stream.query(EventQuery.matchAll());   // throws here, or returns everything
```

`getEventById` throws the same way. And `append` deserializes the events it just wrote in order to return them — so a payload that serializes but cannot be read back fails *there*, as a deserialization failure, with the event already stored.

## Misconfiguration Is IllegalArgumentException

Several registration-time mistakes are properties of the `Class`es handed to `getEventStream(...)`, and fail before anything is read or written:

- `@LegacyEvent` on a class registered as a **current** type
- a current class registered as a **legacy** type
- an upcaster that cannot be instantiated
- an upcaster whose `targetTypes()` names a class the stream does not register, or that the default cannot derive from the declaration (override it)
- upcasters forming a cycle
- an `@EventName` that is blank or padded
- a duplicate stored event name, and a non-sealed interface as a root
- an event type declaring a `Shreddable` on a store with no shredding configured

There is no recovery but to fix the code, so they are not serde failures and no retry applies. The messages name the upcaster *and* the event class and keep the reflective cause.

`Tag.of(...)` and `Tags.of(...)` also reject shapes that would not survive storage — see [tag construction rules](/posts/eventstore-appending-events/#what-a-tag-may-contain) — and `Tags.tag(key)` throws `IllegalStateException` when several tags carry the key.

Queries and appends have their own rules, and violating them throws here too, with nothing read or stored: `or` rejects differing `until` references, differing directions, and any query carrying a limit; a filter naming a legacy type is refused on a stream that upcasts it; an append through a wildcard stream is refused; and a batch repeating an idempotency key is refused.

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

> `ProjectorException.getEventReference()` is the last event **handed to `when`**, which is the failing event only when `when` threw. For a page that could not be read or a batch hook that threw, it is the last event of an earlier batch, or null when there was none. The projector reads a batch as one page, deserialized whole, so a poison event means none of its batch reached the projection: when the cause is a deserialization failure, `EventDeserializationException.getReference()` is the one that names the culprit.
{: .prompt-warning }

A failing batch takes the projector's cursor back to where the batch started, so the same events come round on the next run; a savepoint handler that throws does the same for the run's `initQuery()`.

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
} catch ( EventSerializationException | IdempotencyKeyConflictException e ) {
    // a code bug: nothing was stored, no retry will help
} catch ( EventStorageException e ) {
    // infrastructure: retry with backoff, then fail
}
```

Where it has no policy, let it propagate. An exception that reaches a request boundary and produces a 500 with a stack trace is more useful than one absorbed into a log line.

## Every Exception Survives a Process Boundary

A `Throwable` is `Serializable`, so a field on one that is not makes the whole exception unserializable — and whatever was carrying it across a process boundary (a forked benchmark, a remote test runner, a job scheduler) then reports a `NotSerializableException` **instead of** the failure. Every exception here is therefore kept serializable:

- `EventReference`, `EventId` and `EventType` are `Serializable` records, so `ProjectorException` and `EventDeserializationException` arrive with the event they name intact. Records deserialize through their canonical constructor, so the validation is re-applied on the way in.
- `OptimisticLockingException.getFilter()` reads **null** on a deserialized instance — the filter is a query shape wanted by nobody across a boundary, and the message already names it in text. `getExpectedLastEventReference()` keeps its "never null" contract on the far side.
