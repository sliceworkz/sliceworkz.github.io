---
toc: true
title: Eventstore
description: A DCB-Compliant EventStore
icon: fas fa-solid fa-database
order: 1
---

[![Repo](https://img.shields.io/badge/git_repo-develop-green?logo=github)](https://github.com/sliceworkz/eventstore)
[![Quickstart](https://img.shields.io/badge/Quickstart%20Guide-blue)](/posts/eventstore-quickstart/)
[![Docs](https://img.shields.io/badge/Documentation-purple)](/categories/eventstore-documentation/)


## What is Eventstore?

Sliceworkz Eventstore is an **open source eventstore** implementation in Java.

* If're you're into eventsourcing, have a look at our <a href="/posts/eventstore-quickstart/">quickstart guide</a> and <a href="/categories/eventstore-documentation/">documentation</a> to get started.
* If you're new to eventsourcing, do yourself a favor and <a href="https://leanpub.com/eventmodeling-and-eventsourcing">learn about it</a>

## Features

- Fully compliant with **DCB**, the <a href="https://dcb.events/specification">Dynamic Consistency Boundary specification</a>
- Fully **typed access** to Event via EventStreams, checked by the compiler
- Event **Query** capabilities on event Types and Tags, with a fluent builder and paged reads
- **Optimistic locking** on Event appends, atomic under concurrency, with `head()` to pin a boundary before deciding
- Built-in **upcasting** of legacy events, across chains of versions
- **Stored event names** decoupled from class names with `@EventName`
- **Idempotent appends** scoped per event stream, a batch de-duplicated as a unit
- **Import and migration** between storage backends, preserving event identity
- **Leader election** on named leases, with fencing tokens and priority-based handover
- **GDPR erasure by crypto-shredding** — personal data encrypted per data subject, erased by destroying keys, never by rewriting events
- **Reader entitlement** — services that may not see personal data read the events with it withheld

## Technical

- **Pure Java** implementation
- Lightweight with **minimal dependencies** — the API carries SLF4J and no metrics library; every jar declares an `Automatic-Module-Name`
- **Postgres-based** database storage — **PostgreSQL 16+** supported, with native server-side `uuidv7()` on **18+** and a Java-side fallback on 16–17 (auto-detected at startup)
- **In-Memory storage** for development and unit-testing
- **File-persisted in-memory storage** for local development without PostgreSQL
- **Explicit lifecycle**: storage, store and stream are all `AutoCloseable`
- **Observability through an SPI of its own** — bind Micrometer, OpenTelemetry or anything else with an `EventStoreObserver`
- **Measured performance** — a benchmark suite behind every design choice, from stream layout to the shape of the consistency check
- **Published test module**: a `given/when/then` fixture for applications, and a TCK for custom storage backends

