---
toc: true
title: Eventmodeling
description: An opinionated EventModeling implementation framework
icon: fas fa-solid fa-microchip
order: 2
---

[![Repo](https://img.shields.io/badge/git_repo-develop-green?logo=github)](https://github.com/sliceworkz/eventmodeling)
[![Quickstart](https://img.shields.io/badge/Quickstart%20Guide-blue)](/posts/eventmodeling-quickstart/)
[![Docs](https://img.shields.io/badge/Documentation-purple)](/categories/eventmodeling-documentation/)


## What is EventModeling?

Sliceworkz EventModeling is an **open source, opinionated framework** in Java for building applications designed with <a href="https://eventmodeling.org">Event Modeling</a>: every slice of the event model becomes a **feature slice** in the code, and every box on the model has a home in the framework.

* If you're into event modeling, have a look at our <a href="/posts/eventmodeling-quickstart/">quickstart guide</a> and <a href="/categories/eventmodeling-documentation/">documentation</a> to get started.
* If you're new to event modeling, do yourself a favor and <a href="https://eventmodeling.org">learn about it</a>.

It builds upon the <a href="/eventstore/">Sliceworkz Eventstore</a>, and decides every command within a **Dynamic Consistency Boundary** rather than an aggregate.

## The Event Modeling patterns

| pattern | flow | in the framework |
|---|---|---|
| **State change** | Trigger → Command → Event | `Command`, deciding on `DecisionModel`s |
| **State read** | Events → Read model → UI/API | `ReadModel`, live or eventually consistent |
| **Automation** | Events → To-do list → Processor → Command → Event | `TodoListReadModel` + `Automation` |
| | or: Event → Policy → Command → Event | `Policy` — whenever an event happens, issue a command |
| **Translation** | External event → Processor → Command → Event | `Translator` |

And telling the outside world, in two halves:

| | flow | in the framework |
|---|---|---|
| **Publisher** | Domain event → Outbound event | `Publisher`, mapping recorded facts into the published language |
| **Dispatcher** | Outbound event → external system | `Dispatcher`, the outbox pattern |

## Features

- **Feature slices** discovered by package scanning, each registering its parts per aspect — command, query, automation, projection — so a deployment can run some aspects and not others
- **DCB commands**: the consistency boundary is pinned at the stream head before a command reads, and is the union of everything it decided on — conflicts are detected, never silently admitted
- **Bounded retries** of optimistic-locking conflicts with `executeWithRetry`, and **idempotency keys** for safe resubmission
- **Typed entity ids and tags** — an `AccountId` cannot be passed where a `CustomerId` is expected
- **Business rules with SBVR enforcement levels** — rules people may override under conditions, decided by the command on history, recorded on the events, and **previewable** with `evaluate(...)` before anything is submitted
- **A ladder of read models**: live, savepoint-bounded, eventually consistent in memory (`PublishingReadModel`), durable in SQL, seeded reads that are current without replaying, and snapshots
- **Automations that survive failure**: per-item failure handling, backoff, dead letters as events, at-least-once delivery with the to-do list as the queue
- **Policies**: whenever an event happens, issue a command — redelivery-safe, with an explicit choice of where to start and what a rejection means
- **Publishers and dispatchers**: every outbound event traces back to the domain event it was published for
- **Ports and adapters**, with monitored calls: every call to an external system reported as returned, rejected or failed
- **Leader election**: automations, policies, shared read models, translators, publishers and dispatchers run on exactly one instance of a deployment, with fencing for durable read models
- **Operable**: stop, restart and inspect processors on any instance, locally or through a management stream
- **Capabilities filed by audience** — application code, inbound edge, operator, privacy — so each caller holds only what it needs
- **Observability through its own SPI**: lifecycle events for every component, an observer for metrics and tracing, correlation ids across a whole flow
- **GDPR erasure by crypto-shredding**, configured with one call

## Technical

- **Pure Java 21** implementation, idle processors on virtual threads
- Lightweight with **minimal dependencies** — no metrics library; bind the one you use
- **Postgres-based** event storage — **PostgreSQL 16+** supported, with native server-side `uuidv7()` on **18+** and a Java-side fallback on 16–17, auto-detected at startup
- **In-Memory storage** for development and unit-testing, and **file-persisted in-memory storage** for local development without PostgreSQL
- **SQL-backed read models** with built-in support for H2 and PostgreSQL, idempotent by design
- **Published test module**: given/when/then bases for commands, aggregates, read models, automations, policies, translators, publishers, dispatchers and SQL read models — runnable against every storage backend
