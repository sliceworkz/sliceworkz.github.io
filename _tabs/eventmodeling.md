---
toc: true
title: Eventmodeling
description: An opinionated EventModeling implementation framework
icon: fas fa-solid fa-command
order: 2
---

[![Repo](https://img.shields.io/badge/git_repo-develop-green?logo=github)](https://github.com/sliceworkz/eventmodeling)
[![Docs](https://img.shields.io/badge/Quickstart%20Guide-blue)](https://sliceworkz.github.io/posts/eventmodeling-quickstart/)


## Getting started

Step-by-step introduction with the [quickstart guide](https://sliceworkz.github.io/posts/eventmodeling-quickstart/)

Sliceworkz EventModeling is an **open source opinionated eventmodeling framework** implementation in Java.

* If're you're into eventmodeling, have a look at our <a href="/posts/eventmodeling-quickstart/">quickstart guide</a> to get started.
* If you're new to eventmodeling, do yourself a favor and <a href="https://eventmodeling.org">learn about it</a>

## Features

The EventModeling library provides an opinionated implementation framework to realize Event Modeled applications in Java.

The goal is to maintain a very lightweight implementation of all core patterns involved.

Supports the 4 EM core templates:
- State change	(Trigger -> Command -> Event)
- State read	(Events -> ReadModel -> UI/API
- Automation	(Events -> TODOList -> Processor -> Command -> Event)
- Translation	(External Event -> Processor -> Command -> Event)

And some utility facilities:
- Dispatcher	(Outbound Event outbox pattern)


It builds upon the <a href="/eventstore/">Sliceworkz Eventstore</a> underneath.

## Technical

- **Pure Java** implementation
- Lightweight with **minimal dependencies**
- **Postgres-based** database storage
- **In-Memory storage** for development and unit-testing
