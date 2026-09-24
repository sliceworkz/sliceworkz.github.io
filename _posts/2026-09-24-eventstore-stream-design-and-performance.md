---
layout: post
toc: true
title: Stream Design and Performance
description: A stream per context or a stream per entity, what a read and a consistency check cost, and what the in-memory store cannot tell you — the measured conclusions of the eventstore benchmark suite
date: 2026-09-24 02:00:00
categories: [Eventstore Documentation,Eventstore deployment]
tags: [performance,benchmark,stream design,postgres,dcb,capacity]
---

This guide summarises what the eventstore's benchmark suite measured about the decisions an application author makes early and lives with: how to lay out streams, how to read, and how to write a consistency check that stays cheap. The suite lives in the [`sliceworkz-eventstore-benchmark`](https://github.com/sliceworkz/eventstore/tree/develop/sliceworkz-eventstore-benchmark) module; it runs nothing during a build, publishes every number with a manifest describing where it was measured, and refuses to compare numbers from different environments.

> **Read the figures for direction and rough magnitude.** Unless marked otherwise, they come from PostgreSQL 18 in Testcontainers on a developer machine: good for "clearly faster" and "about ten times", not for a third digit. The ten-million-event figures come from a deliberately configured external server and are published in the repository under `sliceworkz-eventstore-benchmark/results/`.
{: .prompt-info }

## One Stream Per Context, or One Per Entity?

The first layout question: a stream per bounded context, with entities told apart by tags (`inventory/default`, every event tagged `sku:…`), or a stream per entity, with the id as the purpose (`inventory/WIDGET-42`). Both are ordinary uses of `EventStreamId`. The suite measured the same 100.000-event corpus — 2000 entities — laid out both ways, so the difference is attributable to the layout alone.

**A stream per entity wins or ties everything except reading a whole context in order.** Per-entity ÷ per-context, where higher is better for per-entity:

| Workload | 1 thread | 8 threads |
|---|---|---|
| conditional append (one type, one tag — the canonical DCB check) | 4.2× | **16.8×** |
| decide, then append | 1.4× | 2.3× |
| read one entity's history, cold | 2.4× | 2.2× |
| find one needle by tag | 1.5× | 1.4× |
| read one entity's history, hot | 1.2× | 1.2× |
| last event of an entity | 1.1× | 1.2× |
| page a whole context in order (500 events) | 0.074 | 0.066 |
| unconditional append (the control) | 1.04× | 0.98× |

- **The 16.8× on conditional appends is the advisory lock, not an index.** Conditional appends are serialized per stream. With one stream per context, every conditional append in the context takes the *same* lock and eight writers take turns; with one stream per entity, writers to different entities take different locks and never meet. The gap growing with writers is the signature of contention rather than of a cheaper plan.
- **The one loss is reading a context in order** — a whole-context replay, a `Projector` over a context, an export — because under a per-entity layout that is a *cross-entity* read. It was measured before `idx_events_context_order` existed, the index that serves exactly that read by giving it a start condition at the context; on the current schema a page is expected to cost the page.

### Read an Entity Through Its Own Stream, or the Layout Buys Nothing

This is the trap, and it is entirely in the calling code. On a per-entity layout, `EventStreamId.forContext("inventory")` with a wildcard purpose and a tag query addresses one entity *the per-context way* — and lands on no index built for it. Measured on the same per-entity corpus, the "last event of an entity" probe was **23–29× slower** addressed by tag through the wildcard than addressed through the entity's own stream.

```java
// per-entity layout: read the entity through its OWN stream
EventStream<InventoryEvent> widget =
    eventStore.getEventStream(EventStreamId.forContext("inventory").withPurpose("WIDGET-42"), InventoryEvent.class);
List<Event<InventoryEvent>> history = widget.query(EventQuery.matchAll());

// NOT: the whole context, narrowed by tag -- the worst of both layouts
eventStore.getEventStream(EventStreamId.forContext("inventory").anyPurpose(), InventoryEvent.class)
          .query(EventQuery.forTags(Tags.of("sku", "WIDGET-42")));
```

Choosing a stream per entity and then querying by tag is the worst of both.

### A Consistency Boundary Lives Inside One Stream

A stream per entity is a storage layout, not an aggregate — and it has one hard consequence to design for. **The consistency check of a conditional append is scoped to the stream it appends to**: it sees the facts in that stream and no other. Within that stream, the boundary is still drawn with types and tags.

So a rule that spans entities — "a student takes at most five courses, a course at most its capacity" — can only be checked by one append if the facts it depends on are in one stream. With a stream per student and a stream per course, no single append can see both, and the rule cannot be enforced with DCB. Choose the layout per context, from its rules:

- where decisions are **about one entity at a time**, a stream per entity is the faster layout;
- where decisions **span entities**, keep those facts in one stream — per context — and draw the boundary with tags, accepting that conditional appends in that stream share one lock. [Aggregates and DCB](/posts/aggregates-and-dcb/) works through exactly such a rule.

### The Cost the Benchmarks Do Not Include

Every figure here is measured with no [observer](/posts/eventstore-observability-micrometer-prometheus-grafana/). 2000 purposes are 2000 streams to an observer, and one turning the purpose into a metrics tag has to bound it — see [cardinality](/posts/eventstore-observability-micrometer-prometheus-grafana/#cardinality-is-the-observers-concern). That is the observer's cost, measured with the observer.

## Where Writers Meet

Three ways to put writers on one 100.000-event per-entity corpus, differing only in what they share. Conditional appends per millisecond:

| Writers | spread over entities | sharing one stream (lock) | sharing one boundary | …of which useful |
|---|---|---|---|---|
| 1 | 5.45 | 1.30 | 8.15 | 8.15 |
| 4 | 16.98 | 1.36 | 6.40 | 6.09 |
| 8 | 24.06 | 1.39 | 4.66 | 4.02 |
| 16 | 23.59 | 1.34 | 1.24 | **0.22** |

- **A shared lock does not slow an append down; it stops throughput scaling at all.** Writers spread over entities go from 5.45 to 24 — 4.4× — from one to eight; writers sharing one stream's lock stay flat at ~1.4 across a sixteen-fold increase in writers.
- **At one shared boundary, adding writers makes the system strictly worse.** Useful appends fall from 8.15 to 0.22 per ms from one writer to sixteen — a 37× collapse — while the conflict rate climbs from 0% to 82%. Raw throughput hides it: 1.24 ops/ms at sixteen writers still *looks* like work.
- **What to do with it.** A coupon redeemed at most N times, a counter, a single aggregate everyone touches: a boundary that hot has a one-writer ceiling, and more instances behind it buy nothing. **Widen the boundary** — one per basket rather than one per coupon. The lock, by contrast, is bought off by stream layout.

## The Consistency Check, and a Stale Cursor

On PostgreSQL, a conditional append that carries an expected reference checks its boundary with an ordered probe: it walks the stream's position index **forward from that reference** and stops at the first matching event. Its cost is therefore the stream events appended *since the reference*, at about **0.2 µs each** — not the size of the store, and not the size of the entity's history.

That makes the reference you present the whole story:

- **A reference that is the last relevant event of a long-idle entity is stale.** Its last matching event simply *is* old, and every stream event written since has to be walked. Re-reading the boundary does not help: it cannot advance a cursor past events that do not exist. Measured with a boundary pinned half a ten-million-event stream back: ~590 ms per check, perfectly linear.
- **Under ordinary traffic the average walk is about one event per entity active in the stream** — so on a per-context layout the average check prices at ~0.2 µs × the number of entities, whatever the skew and whatever the volume. At ten million events with a realistic mix: ~26 ms per check.
- **Present the head instead.** Take `stream.head()` *before* the decision's read, bound the read with `until(head)`, and hand the head to `AppendCriteria`. The read proved nothing matching sits between the old event and the head, so the head is exactly what the decision may claim — and the probe starts where the reader stopped. Measured at ten million events, a whole decision done this way — two bounded reads plus the checked append — takes **~4 ms**, against **~500 ms** for the same decision with unbounded reads and the last relevant event as the reference. It is also the one conditional write that scales with writers on a per-context stream (3.0× from one writer to eight), because the lock now covers only a sliver of the operation.

The pattern is spelled out in [Optimistic Locking](/posts/eventstore-appending-events/#optimistic-locking). A check with **no** reference — "I decided on an empty boundary", the uniqueness pattern — takes a different path, answered by the tag index at ~2.3 ms at ten million events.

## What a Read Costs

**Deserialization is roughly 2–3 µs per event, and on an ordinary page it is most of the wait.** A 500-event page spends 60–83% of its time in JDBC and deserialization rather than in PostgreSQL. The figure holds across two orders of magnitude: reading one hot entity returns 6.876 events from a 100.000-event store and 455.092 from a ten-million-event one, at about 3 µs per event either way — linear in the rows returned, not in the size of the store. Two consequences:

- **Bounding a read with `EventQuery.limit(n)` is worth more than it looks**, because the cost it bounds is mostly per event and downstream of the query.
- **Tuning the database is the wrong first move** for a read returning thousands of events: the database is not where the time goes.

**Eight of eleven read shapes do not move at a hundred times the volume** — a page, a lookup by id, a query by type, a cold entity, a cursor walk, the last event of an entity, a needle tag query, an OR of facts — measured on the external server at 100.000 and at 10.000.000 events.

**An OR of facts is not served by the tag index.** The GIN index answers one tag containment, not a disjunction of them, so a query of several `or`-ed items walks the stream in order and filters. Fine while the limit fills early; keep the number of OR-ed items small on hot paths.

## Sharing a Table, Sharing a Database

- **Sharing a table with other bounded contexts costs nothing a stream-scoped index can prune.** Ten of twelve read shapes did not move with five other domains at five times the volume in the same table.
- **But a tag's selectivity is a property of the table, not of your context.** The tag index is not stream-scoped: `sku:SKU-000000` matched 6.876 events in one context and 40.227 in the shared table, because other contexts tagged their events with `sku:` too. That flipped an OR-of-facts read from a plan the limit could stop early to one that materialises everything first — 5.4× slower. A limit only bounds work while the plan is one the limit can stop.
- **Sharing only a database with idle neighbour stores costs nothing measurable** — three million-event stores under other prefixes moved nothing. A *busy* neighbour is a different question: a long-running writing transaction anywhere in the cluster freezes what every store can read — see [What Can Stall Reads](/posts/eventstore-configuring-postgresql-storage/#what-can-stall-reads-the-visibility-barrier).

## The In-Memory Store Is Not a Performance Model

The in-memory backends are **correctness**-equivalent to PostgreSQL — the same compliance suite holds both to the same contract, down to what they refuse — and deliberately not performance-equivalent. They hold a list and match in Java: their cost is how far into the log the scan walks, not how many events come back.

At 100.000 events that made the in-memory store **31× slower than PostgreSQL on a needle tag query** and **78× slower reading one long-tail entity's history**, while being 3–90× *faster* where a limit fills immediately — a page, a lookup by id. So an application prototyped against the in-memory store learns nothing about what its tag queries will cost in production, and learns it backwards: selective tag queries are the case PostgreSQL's GIN index exists for, and the case the in-memory store is worst at.

Keep using it for tests and for local development. Size the application against PostgreSQL.

## Summary

- Prefer a **stream per entity** where decisions concern one entity at a time — and then read each entity through its own stream.
- **Keep the facts a decision spans in one stream**: a conditional append checks only the stream it appends to.
- **Take the head before you decide**, bound your reads with it, and present it as the reference.
- **Widen hot boundaries**; more writers on one boundary is strictly worse.
- **Bound reads** with `limit(n)`; deserialization, not the database, is where a large read spends its time.
- **Measure against PostgreSQL**, never against the in-memory store.
