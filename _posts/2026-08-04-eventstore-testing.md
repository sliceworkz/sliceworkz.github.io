---
layout: post
toc: true
title: Testing Your Application
description: The given/when/then fixture, and the compliance suite for custom storage backends
date: 2026-08-04 03:00:00
categories: [Eventstore Documentation,Eventstore API]
tags: [testing,junit,fixture,tck,given when then]
---

This guide covers `sliceworkz-eventstore-testing`, the published module that supports testing code written against the EventStore. It holds two quite different things: a `given / when / then` fixture for **application authors**, and a compliance suite (TCK) for anyone **implementing an `EventStorage`** of their own.

## Adding the Dependency

The module is published like any other and its version is managed by the BOM. Add it in `test` scope:

```xml
<dependency>
    <groupId>org.sliceworkz</groupId>
    <artifactId>sliceworkz-eventstore-testing</artifactId>
    <scope>test</scope>
</dependency>
```

Everything lives in `src/main/java` rather than a test-jar: a test-jar is not transitively resolved and ships no sources or javadoc, which makes it a poor way to distribute a TCK.

| Package | For whom |
|---|---|
| `org.sliceworkz.eventstore.testing.fixture` | application authors — the `given / when / then` fixture |
| `org.sliceworkz.eventstore.testing` | storage authors — the backend harness (`AbstractEventStoreTest`, `EventStoreBackend`, `@ForEachBackend`) |
| `org.sliceworkz.eventstore.testing.tck` | storage authors — the shared compliance scenarios |

## Testing Application Code with EventStoreFixture

Almost every DCB application has the same shape: query the events relevant to a decision, decide, append conditionally. Three parts of that are awkward to test by hand:

- **Seeding history** takes a loop of `append(AppendCriteria.none(), Event.of(...))`
- **Asserting on what was appended** means picking `data()` and `tags()` out of events whose reference and timestamp the store assigns
- **Provoking an `OptimisticLockingException`** deterministically needs an append to land in the window between the decider's query and its own append

`EventStoreFixture` handles all three over a fresh in-memory store — no database, no container, no cleanup.

### The Code Under Test

Nothing in the fixture requires test-specific seams. This is ordinary production-shaped code:

```java
class Registrations {

    private final EventStream<LearningEvent> stream;

    Registrations ( EventStream<LearningEvent> stream ) {
        this.stream = stream;
    }

    boolean subscribe ( String studentId, String courseId ) {
        EventQuery relevant = EventQuery.forEvents(EventTypesFilter.any(), Tags.of("course", courseId));
        List<Event<LearningEvent>> facts = stream.query(relevant).toList();

        // ... decide from the facts ...

        stream.append(
            AppendCriteria.of(relevant, facts.getLast().reference()),
            Event.of(new StudentSubscribed(studentId, courseId),
                     Tags.of("student", studentId, "course", courseId)));
        return true;
    }
}
```

### A First Test

```java
import static org.sliceworkz.eventstore.testing.fixture.ExpectedEvent.event;

class SubscribeStudentToCourseTest {

    private final EventStoreFixture<LearningEvent> fixture =
        EventStoreFixture.inMemory(EventStreamId.forContext("learning"), LearningEvent.class);

    @Test
    void subscribesAStudentToACourseWithCapacity ( ) {
        fixture.given(event(new CourseDefined("abc001", "Java basics", 12)).tagged("course", "abc001"))
               .when(stream -> new Registrations(stream).subscribe("123", "abc001"))
               .expectResult(true)
               .expectAppended(event(new StudentSubscribed("123", "abc001"))
                   .tagged("student", "123")
                   .tagged("course", "abc001"));
    }

    @Test
    void studentCannotSubscribeTwice ( ) {
        fixture.given(
                   event(new CourseDefined("abc001", "Java basics", 12)).tagged("course", "abc001"),
                   event(new StudentSubscribed("123", "abc001"))
                       .tagged("student", "123").tagged("course", "abc001"))
               .when(stream -> new Registrations(stream).subscribe("123", "abc001"))
               .expectResult(false)
               .expectNoEventsAppended();
    }
}
```

> A fixture is single-use per test. Build a new one per test method — a field initialiser is enough, since JUnit creates a fresh test instance per method — so history never leaks between tests.
{: .prompt-warning }

### Creating a Fixture

```java
// a fresh in-memory store, on the stream you name
EventStoreFixture.inMemory(EventStreamId.forContext("learning"), LearningEvent.class);

// same thing, on a stream in context "test"
EventStoreFixture.inMemory(LearningEvent.class);

// over a storage you provide — a PostgreSQL store, for instance.
// the fixture does not own it and will not close it
EventStoreFixture.over(myStorage, EventStreamId.forContext("learning"), LearningEvent.class);
```

`fixture.stream()`, `fixture.eventStore()` and `fixture.eventStorage()` expose the objects underneath when a test needs to step outside the fluent API.

### Seeding History

`given(...)` appends the events unconditionally, in order. `ExpectedEvent.event(...)` is a payload plus tags — deliberately not a full `Event`, because reference and timestamp are the store's business:

```java
fixture.given(
           event(new CourseDefined("abc001", "Java basics", 12)).tagged("course", "abc001"),
           event(new StudentSubscribed("1", "abc001")).tagged("course", "abc001"))
       .and(event(new StudentSubscribed("2", "abc001")).tagged("course", "abc001"))
       ...
```

Use `givenNoHistory()` for an empty stream, and `lastReference()` to capture the reference of the last seeded event — useful as a boundary for point-in-time projection tests:

```java
EventReference afterTheSecondSubscription = fixture.given(
        event(new CourseDefined("abc001", "Java basics", 12)).tagged("course", "abc001"),
        event(new StudentSubscribed("1", "abc001")).tagged("course", "abc001"),
        event(new StudentSubscribed("2", "abc001")).tagged("course", "abc001"))
    .lastReference();
```

### Running the Decision

`when(...)` runs a decider that returns a value; `whenRunning(...)` runs one that returns nothing:

```java
.when(stream -> new Registrations(stream).subscribe("123", "abc001"))     // returns boolean
.whenRunning(stream -> new Registrations(stream).cancelCourse("abc001"))  // returns void
```

Both hand back a `DeciderOutcome` carrying what the decider appended, what it returned, and — if it threw — the exception. Nothing is rethrown at that point, so a test can assert on a failure instead of catching it.

### Asserting the Outcome

```java
outcome.expectAppended(event(...), event(...));   // exact payloads and tags, in order
outcome.expectNoEventsAppended();
outcome.expectResult(false);                      // the decider's return value
outcome.expectResult(r -> r > 3, "more than three");
outcome.expectNoFailure();
outcome.expectFailure(IllegalStateException.class);   // returns the exception, for further asserting
```

`appended()`, `result()` and `failure()` give raw access when the assertions above are not enough.

> Assertions compare an event's **payload and tags only**. Stream, reference and timestamp are assigned by the storage and cannot be predicted from a test: the in-memory store stamps events from the JVM clock, the PostgreSQL store lets the server clock do it, and there is no clock seam in either. Assert on timestamps with a tolerance window or not at all.
{: .prompt-info }

Assertion failures explain themselves in terms of what actually happened. Asserting on appends after a decider threw reports *the decision threw IllegalStateException*, rather than an unhelpful "expected 0 events, got 0".

### Provoking a DCB Conflict

This is the part that is genuinely hard to write by hand. `whenConcurrently(...)` appends the interleaved events into the window between the decider's query and its own append — the only deterministic way to make the consistency boundary fire:

```java
@Test
void anInterleavedSubscriptionMakesTheBoundaryFire ( ) {
    fixture.given(event(new CourseDefined("abc001", "Java basics", 12)).tagged("course", "abc001"))
           .whenConcurrently(
               stream -> new Registrations(stream).subscribe("123", "abc001"),
               event(new StudentSubscribed("999", "abc001"))
                   .tagged("student", "999").tagged("course", "abc001"))
           .expectOptimisticLockingFailure()
           .matchingTags("course", "abc001");
}
```

`expectOptimisticLockingFailure()` returns an `OptimisticLockingFailure` that lets you assert on the boundary that fired, not merely that something did:

```java
.expectOptimisticLockingFailure()
    .matchingTags("course", "abc001")            // the filter the append was conditional on
    .expectedLastReference(someReference)         // the reference the decision was taken from
    .expectedAnEmptyBoundary();                   // the decision was taken on an empty result
```

Equally important is the negative test — that an unrelated event does **not** break the append. This is where DCB earns its keep over stream-level locking, and it deserves a test:

```java
@Test
void anInterleavedAppendOutsideTheBoundaryDoesNotFire ( ) {
    fixture.given(event(new CourseDefined("abc001", "Java basics", 12)).tagged("course", "abc001"))
           .whenConcurrently(
               stream -> new Registrations(stream).subscribe("123", "abc001"),
               event(new CourseDefined("zzz999", "Unrelated", 5)).tagged("course", "zzz999"))
           .expectNoFailure()
           .expectResult(true);
}
```

### Driving a Projection

`project(...)` runs a `Projection` over the seeded history, optionally up to a boundary:

```java
@Test
void countsOnlySubscriptionsUpToTheBoundary ( ) {
    EventReference upToTheSecond = fixture.given(
            event(new CourseDefined("abc001", "Java basics", 12)).tagged("course", "abc001"),
            event(new StudentSubscribed("1", "abc001")).tagged("course", "abc001"),
            event(new StudentSubscribed("2", "abc001")).tagged("course", "abc001"))
        .lastReference();

    fixture.given(event(new StudentSubscribed("3", "abc001")).tagged("course", "abc001"))
           .project(new SubscriptionCount("abc001"))
           .upTo(upToTheSecond)
           .expectEventsProcessed(2)
           .expectState(count -> assertEquals(2, count.count()));
}
```

Without `upTo(...)` the projection runs to the head of the store. `inBatchesOf(n)` sets the projector's batch size, `projection()` returns the projection itself and `metrics()` the `ProjectorMetrics` of the run.

## Testing a Custom EventStorage: the TCK

If you implement `EventStorage` against a backend of your own, the same module carries the compliance suite the in-tree backends are held to. Running it is one dependency and one line of surefire configuration.

### Declaring Your Backend

Implement `EventStoreBackend` — a name, how to create a storage, and which optional capabilities you support:

```java
public class MyBackend implements EventStoreBackend {

    @Override
    public String name ( ) {
        return "mystorage";
    }

    @Override
    public EventStorage createEventStorage ( StorageOptions options ) {
        return MyEventStorage.newBuilder()
            .prefix(options.prefix())
            .resultLimit(options.resultLimit())
            .build();
    }

    @Override
    public boolean supports ( Capability capability ) {
        return capability != Capability.IMPORT;   // everything except importEvents
    }
}
```

Backends are discovered with the `ServiceLoader`, so register it:

```
src/test/resources/META-INF/services/org.sliceworkz.eventstore.testing.EventStoreBackend
```
```
com.example.MyBackend
```

`destroyEventStorage(storage)` defaults to `storage.close()` — the SPI contract already requires that to release everything the storage created and to block until it has, so override it only to release something *you* handed the storage, such as a pool it deliberately will not close.

### Running the Suite

The TCK scenarios are main classes of the testing module, not test classes of your project, so surefire has to be told to scan that artifact:

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <configuration>
        <dependenciesToScan>
            <dependency>org.sliceworkz:sliceworkz-eventstore-testing</dependency>
        </dependenciesToScan>
    </configuration>
</plugin>
```

Every scenario now runs against your backend. Among what it pins down: basic append and query semantics, tag round-tripping over the full legal character set, the `until` boundary, query limits, concurrent optimistic locking, append-notification granularity, subscription and storage lifecycle, serde failure reporting, and — where supported — importing.

### Capabilities

Not every backend supports every optional part of the contract. Scenarios that need one declare it, and are **skipped rather than failed** where it is absent:

| Capability | Meaning |
|---|---|
| `IMPORT` | `importEvents(...)` is implemented |
| `TABLE_PREFIX` | several independent stores can coexist, kept apart by a prefix |
| `RESULT_LIMIT` | a store can be built with an absolute cap on query result size |
| `RAW_STORAGE_ACCESS` | a test can reach the underlying storage directly (e.g. a `DataSource`) |

### Writing Your Own Scenarios

Your own tests extend `AbstractEventStoreTest` and annotate scenarios `@ForEachBackend` instead of `@Test`, so each runs once per registered backend and is reported under its own name (`testQueryOneEvent [mystorage]`):

```java
class MyStorageQuirksTest extends AbstractEventStoreTest {

    @ForEachBackend
    void appendsAndReadsBack ( ) {
        EventStream<CustomerEvent> stream =
            eventStore().getEventStream(EventStreamId.forContext("customer"), CustomerEvent.class);

        stream.append(AppendCriteria.none(),
                      Event.of(new CustomerRegistered("John"), Tags.none()));

        assertEquals(1, stream.query(EventQuery.matchAll()).count());
    }

    @ForEachBackend(requires = Capability.IMPORT)
    void importsEventsPreservingIds ( ) {
        // skipped, not failed, on a backend without IMPORT
    }
}
```

`AbstractEventStoreTest` provides:

- `eventStore()` / `eventStorage()` — the store under test, fresh and empty per test method
- `storageOptions()` — override to ask the backend for a store with a result limit or a table prefix
- `createEventStorage()` — override to supply a storage directly instead of going through a backend
- `waitBecauseOfEventualConsistency(BooleanSupplier)` — an Awaitility helper for asynchronous listener assertions
- `dataSource()` — direct database access, where the backend is SQL-backed

`@ForEachBackend(excludingBackends = "...")` opts a backend out for **cost**, not capability — a scenario too slow to run against a particular store. It is also reported as skipped, so the gap stays visible.

### Narrowing a Local Run

A full run starts every registered backend, which for containerised ones is most of the wall-clock time. Restrict it while iterating:

```bash
mvn test -Deventstore.testing.backends=mystorage
```

The same property partitions a CI run across separate JVMs, which is the supported way to parallelise: in-JVM parallelism is not an option while per-test isolation works by dropping and recreating a store's objects, since two scenarios sharing a backend concurrently would tear down each other's state mid-test.

## What Cannot Be Asserted

**Timestamps.** Events are stamped by the JVM clock in memory and by the server clock on PostgreSQL, with no `Clock` seam in either. Assert on them with a tolerance window, or not at all. The one path that writes a chosen timestamp is `importEvents`, which bypasses `append()` — see [Importing Events Between Stores](/posts/eventstore-importing-events/).

**Positions as absolute numbers.** A position is unique across a storage, not per stream, so a test that seeds two streams cannot predict the numbers. Compare references with `happenedBefore` / `happenedAfter` instead of comparing positions.

**Notification timing.** Append listeners are eventually consistent by construction. Use `waitBecauseOfEventualConsistency(...)` rather than a sleep, and never assert that a projection has *not yet* advanced.
