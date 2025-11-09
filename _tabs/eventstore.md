---
toc: true
title: Eventstore
description: A DCB-Compliant EventStore
icon: fas fa-solid fa-database
order: 1
---
## What is Eventstore?

Sliceworkz Eventstore is an **open source eventstore** implementation in Java.

* If're you're into eventsourcing, have a look at our <a href="/posts/eventstore-quickstart/">quickstart guide</a> to get started.
* If you're new to eventsourcing, do yourself a favor and <a href="https://leanpub.com/eventmodeling-and-eventsourcing">learn about it</a>

## Features

- Fully compliant with **DCB**, the <a href="https://dcb.events/specification">Dynamic Consistency Boundary specification</a>
- Fully **typed access** to Event via EventStreams
- Event **Query** capabilities on event Types and Tags
- **Optimistic locking** on Event appends
- Built-in **upcasting** of legacy events

## Technical

- **Pure Java** implementation
- Lightweight with **minimal dependencies**
- **Postgres-based** database storage
- **In-Memory storage** for development and unit-testing

