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
- Fully **typed access** to Event via EventStreams
- Event **Query** capabilities on event Types and Tags
- **Optimistic locking** on Event appends, atomic under concurrency
- Built-in **upcasting** of legacy events
- **Idempotent appends** scoped per event stream
- **Import and migration** between storage backends, preserving event identity
- **Leader election** on named leases, with fencing tokens and priority-based handover
- **GDPR erasure by crypto-shredding** — personal data encrypted per data subject, erased by destroying keys, never by rewriting events

## Technical

- **Pure Java** implementation
- Lightweight with **minimal dependencies**
- **Postgres-based** database storage — **PostgreSQL 16+** supported, with native server-side `uuidv7()` on **18+** and a Java-side fallback on 16–17 (auto-detected at startup)
- **In-Memory storage** for development and unit-testing
- **File-persisted in-memory storage** for local development without PostgreSQL
- **Explicit lifecycle**: storage, store and stream are all `AutoCloseable`
- **Micrometer metrics** with bounded meter cardinality
- **Published test module**: a `given/when/then` fixture for applications, and a TCK for custom storage backends

