---
layout: post
toc: true
title: Leader Election with Leases
description: Electing one processor among several instances, with fencing tokens and priority-based handover
date: 2026-08-07 01:00:00
categories: [Eventstore Documentation,Eventstore API]
tags: [leader election,lease,fencing,high availability,eventstorage]
---

This guide covers the lease operations on `EventStorage` — the primitive a framework or an application builds leader election on, so exactly one instance of a deployment processes a stream while the others stand by.

## What a Lease Is For

A [bookmark](/posts/eventstore-bookmarking/) says *where* a reader got to. It does not say *which* instance is entitled to move it. Three replicas of the same service all holding the reader name `order-dispatcher` will each happily project and each happily bookmark, and the work that must not happen twice — publishing to an external system, sending a mail, calling a payment provider — happens three times.

A **named lease** answers the other half: one owner holds it, everybody else stands by, and the holder is the only legitimate processor until it releases the lease or lets it expire.

```java
EventStorage storage = PostgresEventStorage.newBuilder().build();

LeaseResponse response = storage.requestLease(
    new LeaseRequest("order-dispatcher",     // the lease name, globally unique in the storage
                     "pod-7f3a9",            // this instance's identity
                     0,                      // priority
                     Duration.ofSeconds(30)) // time-to-live
);

if (response.status() != LeaseStatus.STANDBY) {
    dispatchPendingOrders(response.fencingToken());
}
```

Leases live on the **storage**, not on the `EventStore` or an `EventStream` — like [importing](/posts/eventstore-importing-events/), this is infrastructure rather than domain. If you built your store with `buildStore()`, keep a reference to the storage as well when you intend to elect on it:

```java
EventStorage storage = PostgresEventStorage.newBuilder().build();
EventStore eventStore = EventStoreFactory.get().eventStore(storage);
```

Everything involved lives in the API module, so no extra dependency is needed: `Lease` in `org.sliceworkz.eventstore.events`, and `LeaseRequest`, `LeaseResponse` and `LeaseStatus` nested in `EventStorage` itself (`org.sliceworkz.eventstore.spi`).

## One Call Does Everything

`requestLease(...)` is acquisition, renewal **and** contender registration in a single call. Every instance — leader and standby alike — simply calls it on a timer and reads the answer:

| Status | What it means | What to do |
|---|---|---|
| `LEADER` | you hold the lease and are the single legitimate processor | process |
| `LEADER_STEP_DOWN_REQUESTED` | you still hold it, but a live contender with a **strictly higher** priority is waiting | finish the current unit of work, then `releaseLease` |
| `STANDBY` | someone else holds a live lease; you are registered as a contender | do nothing, keep requesting |

There is no separate "join the election" call, and no watch to register. A standby is simply an instance whose last request was recent enough to count as live.

```java
public class DispatcherLoop {

    private static final String LEASE = "order-dispatcher";
    private static final Duration TTL = Duration.ofSeconds(30);

    private final EventStorage storage;
    private final String me;          // stable for this instance's lifetime, unique among instances
    private final long priority;

    void tick() {
        LeaseResponse response = storage.requestLease(
            new LeaseRequest(LEASE, me, priority, TTL));

        switch (response.status()) {
            case LEADER -> dispatchPendingOrders(response.fencingToken());

            case LEADER_STEP_DOWN_REQUESTED -> {
                dispatchPendingOrders(response.fencingToken());   // finish this unit of work
                storage.releaseLease(LEASE, me);                  // then hand over
            }

            case STANDBY -> { /* nothing to do */ }
        }
    }
}
```

Call `tick()` on a schedule of roughly **a third of the ttl** — that leaves two failed attempts before leadership lapses.

> The `owner` string must be stable for the instance's lifetime and unique among contenders. A pod name, a container id, or a hostname plus a start-up UUID all work; a value that changes on every request turns one instance into an endless stream of new contenders.
{: .prompt-warning }

## Expiry Is Judged on the Storage's Clock

A lease is live while its last heartbeat is younger than the ttl it was requested with, **measured on the storage's clock** — never on any contender's. Contenders' clocks therefore need not agree with each other or with the database: they only ever measure *durations* on their own clock (their polling interval, and the time since their last confirmed renewal), never compare instants across machines.

The guarantee a caller gets: between two calls that both returned `LEADER` (or `LEADER_STEP_DOWN_REQUESTED`), no other owner has held the lease — **provided** the caller stops acting as leader the moment it can no longer *confirm* a renewal within the ttl it asked for.

```java
LeaseResponse response;
try {
    response = storage.requestLease(new LeaseRequest(LEASE, me, priority, TTL));
} catch (EventStorageException e) {
    // a renewal that failed or hung is not evidence of still being the leader
    stopProcessing();
    return;
}
```

That half is the caller's job and cannot be moved into the store: storage-clock expiry plus self-demotion on the caller's clock is what keeps two leaders from overlapping. The one case neither can prevent is a process paused past its own ttl — a long GC pause, a suspended VM — which is exactly what the fencing token is for.

## Fencing Tokens

Every response carries the lease's current fencing token: the caller's own when it is the leader, the current holder's when it is standing by.

- It **strictly increases on every change of ownership**, starting at 1 for the first owner
- It is **stable across renewals** by the same owner
- It **never resets** — releasing a lease does not delete it, it backdates the heartbeat, precisely so the token survives

Stamp outgoing work with the token, and a downstream store can reject anything arriving with a token lower than the highest it has already seen. That is how a zombie leader — one that was paused past its ttl and woke up still believing it holds the lease — is recognised rather than merely hoped against.

```java
void dispatchPendingOrders(long fencingToken) {
    // the receiving side keeps the highest token it has accepted for this lease
    // and refuses anything older, so a superseded leader's writes are rejected
    downstream.publish(batch, fencingToken);
}
```

## Priority and Planned Handover

Priority is what makes handover *deliberate* rather than a race won by whoever happens to poll first.

A renewal turns into `LEADER_STEP_DOWN_REQUESTED` as soon as a **live** contender with a **strictly higher** priority exists. Equal or lower priorities never trigger it. The storage never revokes a live lease itself — a step-down is always the holder's own act, and a leader that cannot stop safely may keep renewing and keep the lease.

Useful shapes for this:

- **Zone or region preference** — the instance closest to the data gets the higher priority, so it takes over as soon as it is healthy
- **Draining a node** — raise the priority of the replacement instance and the outgoing one is asked to finish and hand over, instead of the work stopping for a whole ttl
- **Deploy ordering** — the newly rolled-out instance outranks the old one, so leadership follows the deploy

```java
// nudge the current leader into handing over, without killing it
long priority = runningInPreferredZone() ? 10 : 0;
```

Without a step-down, a takeover still happens — it just waits out the ttl or a voluntary release.

## Releasing

```java
storage.releaseLease("order-dispatcher", me);
```

A release makes the lease immediately acquirable, so the next request by any contender acquires it instead of waiting out the ttl. It also withdraws the owner's contender registration.

It is idempotent and forgiving: releasing a lease you do not hold — because it expired and was taken over, or was never acquired — does nothing and does not throw. A release never touches a lease held by a *different* owner. Release on graceful shutdown; it is the difference between a rolling restart that pauses for milliseconds and one that pauses for a ttl.

## Inspecting Leases

`getLeases()` returns a snapshot of every lease the storage records, including expired ones that nobody has taken over yet:

```java
public record Lease (
    String leaseName,
    String owner,
    long priority,
    long fencingToken,
    Instant acquiredAt,     // storage clock
    Instant heartbeatAt,    // storage clock
    Duration ttl
) {
    public boolean isExpiredAt ( Instant now ) { /* ... */ }
}
```

An expired lease keeps its last owner and heartbeat until someone else acquires it, so liveness is judged against `heartbeatAt` — not against presence in the list:

```java
storage.getLeases().forEach(lease -> LOGGER.info("{}: {} (token {}), last heartbeat {}",
        lease.leaseName(), lease.owner(), lease.fencingToken(), lease.heartbeatAt()));
```

`isExpiredAt(now)` expects an instant from the **same clock** that produced `heartbeatAt` — the storage's. Passing `Instant.now()` from an application server compares two clocks, which is exactly what leases exist to avoid; treat that as a monitoring approximation, never as a decision to start processing.

Like bookmarks, leases are addressed globally by name, so the list spans the whole storage rather than any one stream.

## Backend Support

The lease methods are **optional** on the SPI: their defaults throw `UnsupportedOperationException`, and the TCK gates its lease scenarios on `Capability.LEASE` so a backend written before leases existed skips them rather than failing them. See [Testing](/posts/eventstore-testing/#capabilities).

| Backend | Leases |
|---|---|
| PostgreSQL | yes — two coordination tables outside the event log |
| In-memory | yes — real contention between contenders within one storage instance |
| File-persisted in-memory | yes, but **not persisted** — a lease held by a process that no longer runs must expire, not be resurrected on reload |

Because the in-memory backends contend for real, a test can elect between two contenders without a database — while a single process trivially wins everything it asks for.

### On PostgreSQL

Leases are `<prefix>leases` and `<prefix>lease_contenders`, written in one short transaction on the ordinary pool and serialized per lease by an advisory lock. What that buys, spelled out:

- Election traffic **never touches the events table** and takes no lock that any query or append takes
- A waiting contender holds no transaction id, and lease writes are milliseconds — so leases neither pin `pg_snapshot_xmin` nor are held up by it. This is also why a lease is deliberately **not** modelled as events: event reads sit behind the xmin barrier, and one long writing transaction anywhere in the cluster would otherwise make every lease look expired at once
- All timestamps are written and compared with `now()` in SQL, so the database is the single clock

Both tables are created by `ENSURE` and checked by `VALIDATE`, and the leases table needs no `DELETE` privilege. A deployment pinned to `VALIDATE` or `NONE` has to have them applied from the shipped DDL, like every other object in the schema. See [Configuring PostgreSQL Storage](/posts/eventstore-configuring-postgresql-storage/#the-lease-tables).

## Putting It Together with Bookmarks

Leader election and bookmarking answer different questions, and a resilient processor uses both: the lease decides *who* processes, the bookmark decides *from where*.

```java
void tick() {
    LeaseResponse response = storage.requestLease(new LeaseRequest(LEASE, me, priority, TTL));
    if (response.status() == LeaseStatus.STANDBY) {
        return;                                  // someone else owns this reader right now
    }

    projector.readBookmark();                    // resume where the previous owner left off
    projector.run();                             // bookmarks per batch as it goes

    if (response.status() == LeaseStatus.LEADER_STEP_DOWN_REQUESTED) {
        storage.releaseLease(LEASE, me);         // the next owner picks up from the bookmark
    }
}
```

Reading the bookmark **after** winning the lease, rather than caching it at startup, is what makes a takeover correct — the previous owner moved it. That is what `readBeforeEachExecution()` does for a bookmarked projector; see [Bookmark Read Frequencies](/posts/eventstore-projecting-events/#bookmark-read-frequencies).

Note what is *not* claimed here: the lease does not make processing exactly-once. A takeover after a committed batch whose bookmark did not land re-projects that batch, as it always would. Where that matters, the projection holds its own position — see [Being Exactly-Once Against Your Own Store](/posts/eventstore-projecting-events/#being-exactly-once-against-your-own-store).
