---
layout: post
toc: true
title: Ports, Adapters and Monitored Calls
description: How a feature slice reaches outside its bounded context through a port the application binds, and how every call through it is timed, sorted into an answer or a failure, and attributed to the slice that made it.
date: 2026-10-11 10:00:00
categories: [Eventmodeling Documentation,Eventmodeling Guides]
tags: [eventmodeling, ports, adapters, monitoring, tracing, hexagonal architecture]
---

A feature slice decides on events, but sooner or later one of them has to talk to something that is not an event: a payment network, a customer register run by another team, a message bus, the HTTP router the application uses. The framework's answer is a **port**: an interface the slice asks for, and an **adapter** the application binds to it on the builder.

That split puts each decision where it belongs:

- **The slice keeps its wiring.** It knows it needs a `TransferGateway` and constructs its own automation around one. Nothing outside the slice repeats that wiring.
- **The application keeps the choice of infrastructure.** A real network client in production, a simulated gateway in a demo, a scripted fake in a test — the slice cannot tell and does not need to.
- **A missing adapter fails the build**, not the first transfer in production.

On top of that, a port can be **monitored**: every call is timed, sorted into one of three outcomes, and reported with the component and the slice that made it, in the same flow as the request that caused it.

This guide uses the banking domain of the [quickstart](/posts/eventmodeling-quickstart/), with the outgoing transfers of [Automations and Policies](/posts/eventmodeling-automations-and-policies/).

> The reference in the repository is the payments example: [`PaymentGateway`](https://github.com/sliceworkz/eventmodeling/blob/develop/sliceworkz-eventmodeling-examples/src/main/java/org/sliceworkz/eventmodeling/examples/payments/features/executepayment/PaymentGateway.java) is the port, [`ExecutePaymentFeatureSlice`](https://github.com/sliceworkz/eventmodeling/blob/develop/sliceworkz-eventmodeling-examples/src/main/java/org/sliceworkz/eventmodeling/examples/payments/features/executepayment/ExecutePaymentFeatureSlice.java) takes it, and [`PaymentsExample`](https://github.com/sliceworkz/eventmodeling/blob/develop/sliceworkz-eventmodeling-examples/src/main/java/org/sliceworkz/eventmodeling/examples/payments/PaymentsExample.java) binds a monitored adapter to it. The banking code below has the same shape.
{: .prompt-info }

## Declaring a Port

A port is a plain Java interface. Write it in the words of the slice that needs it, not of the system behind it, and say in its exceptions **whose problem a failure is**:

```java
package com.example.banking.features.executetransfer;

/** The payment network a transfer leaves the bank through. */
public interface TransferGateway {

    /**
     * @return the gateway's own reference for the transfer
     * @throws TransferDeclinedException the gateway declined this transfer, for now: an answer
     * @throws TransferRejectedException this transfer will never be accepted: an answer
     * @throws GatewayUnavailableException the gateway cannot be reached: nothing to do with this transfer
     */
    String execute ( String toIban, long amountInCents, String idempotencyKey );

    class TransferDeclinedException extends RuntimeException {
        public TransferDeclinedException ( String message ) { super(message); }
    }

    class TransferRejectedException extends RuntimeException {
        public TransferRejectedException ( String message ) { super(message); }
    }

    class GatewayUnavailableException extends PortUnavailableException {
        public GatewayUnavailableException ( String message ) { super(message); }
    }
}
```

Three kinds of ending, and they want opposite responses:

| a call that | means | example |
|---|---|---|
| returns | it worked | the gateway's reference |
| throws a **business exception** | the port answered "no": declined, not found, not allowed | `TransferDeclinedException` |
| throws anything else | something is wrong: unreachable, timed out, a bug in the adapter | `GatewayUnavailableException`, a `NullPointerException` |

A **`PortUnavailableException`** (package `org.sliceworkz.eventmodeling.ports`) is the one failure type the framework knows by name: the port could not be called at all. Let the adapter's own "cannot reach" exception extend it, and one type serves every consumer of that failure: an automation's `onFailure` matches it to retry rather than stop, an HTTP binding maps it to `503 Service Unavailable`, and a monitored port always counts it as a failure.

> A business exception is either an `org.sliceworkz.eventmodeling.commands.BusinessException` — the same type a command throws for a rule — or a type you declare on the binding (below). The second is how a third-party client's exceptions become answers without wrapping them.
{: .prompt-tip }

## Binding an Adapter

The application binds an adapter to each port on the bounded context builder, `.adapter(x).forPort(Port.class)`:

```java
Banking banking = BoundedContext.newBuilder(Banking.class)
    .name("banking")
    .eventStorage(storage)
    .instance(InstanceFactory.determine("banking-app"))
    .listener(monitoringListener)
    .adapter(routes).forPort(HttpRoutes.class)                       // configuration: not monitored
    .adapter(customerDirectory).monitored().forPort(CustomerDirectory.class)
    .adapter(gateway)
        .monitored(PortMonitoring.perCall()
                .businessExceptions(TransferDeclinedException.class, TransferRejectedException.class))
        .forPort(TransferGateway.class)
    .adapter(messageBus)
        .monitored(PortMonitoring.summarized(Duration.ofMinutes(1)))
        .forPort(MessageBus.class)
    .features().rootPackage(Banking.class.getPackage()).done()
    .build();
```

`forPort` checks that the adapter is assignable to the port type and throws an `IllegalArgumentException` when it is not. `.monitored(...)` is optional and sits between `adapter(...)` and `forPort(...)`; the rest of this guide is about it.

### Qualified Bindings

One port type can be bound more than once, under a **qualification**. The slice asks for the one it means:

```java
.adapter(euBus).monitored().forPort(MessageBus.class, "eu")
.adapter(usBus).forPort(MessageBus.class, "us")

MessageBus bus = builder.port(MessageBus.class, "eu");
```

`port(MessageBus.class)` without a qualification asks for the default binding, the one made with `forPort(MessageBus.class)`. A monitored call through a qualified binding carries the qualification on its events.

## Taking a Port in a Slice

### While Configuring: `builder.port(...)`

A slice that constructs a component around a port takes the port from the builder in its `configure...` hook:

```java
@FeatureSlice(chapter = "Transfers")
public class ExecuteTransferFeatureSlice implements Slice<Banking> {

    @Override
    public void configureAutomation ( BoundedContextBuilder<Banking> builder ) {
        TransferGateway gateway = builder.port(TransferGateway.class);
        var todoList = new TransfersToExecuteTodoList();
        builder.readmodel(todoList).eventuallyConsistent();
        builder.automation(new ExecuteTransferAutomation(todoList, gateway));
    }
}
```

`builder.port(...)` hands out whatever the binding produces: the adapter itself for an unmonitored binding, the reporting proxy for a monitored one. The slice cannot tell and does not need to. Dispatchers take their outbound channel the same way:

```java
@Override
public void configureAutomation ( BoundedContextBuilder<Banking> builder ) {
    builder.publisher(new AccountAnnouncedPublisher())
           .dispatcher(new AccountAnnouncedDispatcher(builder.port(MessageBus.class)));
}
```

### A Missing Adapter Fails the Build

The `configure...` hooks run inside `build()`. A slice asking for a port nobody bound makes `build()` throw, before anything has started:

```
java.lang.RuntimeException: Failed to register annotated class: com.example.banking.features.executetransfer.ExecuteTransferFeatureSlice
Caused by: java.lang.IllegalStateException: No adapter registered for port 'com.example.banking.features.executetransfer.TransferGateway' with qualification default
```

The message names the slice; its cause names the port. Either way it surfaces at startup — not as the automation failing on its first transfer.

A slice is only configured for the aspects an instance runs, and that works in your favour here: an instance that does not deploy automations never calls `configureAutomation`, never asks for the `TransferGateway`, and need not bind one. A front-end process serving commands and queries can leave the gateway out entirely:

```java
.features().rootPackage(Banking.class.getPackage()).disableAutomations().done()
```

> The alternative — the application constructing the automation itself and registering it beside the scanned slices — loses because the slice is then a slice in name only: its wiring lives in another file, and every deployment has to repeat it.
{: .prompt-info }

### While Starting: `context.port(...)`

A slice wires what runs *beside* its components — an HTTP endpoint, a scheduler — in its `start...` hooks. The context it is handed there is your context interface, so the endpoint takes its ports from it and holds the narrow `BankingApi` for everything else (see [Who May Do What](/posts/eventmodeling-who-may-do-what/)):

```java
@FeatureSlice(chapter = "Account management")
public class OpenAccountFeatureSlice implements Slice<Banking> {

    @Override
    public void configureCommand ( BoundedContextBuilder<Banking> builder ) {
        builder.command(OpenAccountCommand.class);
    }

    @Override
    public void startCommand ( Banking banking ) {
        BankingApi api = banking;                                             // what the endpoint executes through
        CustomerDirectory directory = banking.port(CustomerDirectory.class);  // reported as this slice's call
        banking.port(HttpRoutes.class).post("/accounts", request -> {
            CustomerId customerId = CUSTOMER.id(request.get("customerId"));
            directory.verify(customerId);
            return api.execute(new OpenAccountCommand(customerId));
        });
    }
}
```

Two ports, two roles. `HttpRoutes` is configuration — the application's router, an interface of the application's own (`void post(String path, Function<Map<String,String>, Object> handler)` here), handed to the slice so it can register its route. `CustomerDirectory` is a dependency the endpoint calls on every request, so that is the one worth monitoring.

The endpoint may also look the port up per request, `banking.port(CustomerDirectory.class).verify(...)` inside the handler: either way its calls are attributed to the slice (see [Who Called](#who-called) below).

### Where a Shared Port Lives

A port used by one slice lives in that slice's package. A port two slices use — `CustomerDirectory`, here both opening an account and a customer check — lives **outside** the feature packages, in a package of its own such as `com.example.banking.ports`. Slices do not import from each other, and the published architecture test enforces it:

```
Class com.example.banking.features.checkcustomer.CheckCustomerFeatureSlice depends on
    com.example.banking.features.openaccount.CustomerDirectory from different feature package
```

See [Testing Feature Slices](/posts/eventmodeling-testing/#architecture-rules) for that test.

## Monitored Ports

### `.monitored()` and `PortMonitoring`

A monitored binding hands the port out as a `java.lang.reflect.Proxy` that times every call and reports how it ended. An unmonitored binding hands out the adapter itself, untouched.

| binding | reports |
|---|---|
| `.monitored()` | every call as it ends — the same as `.monitored(PortMonitoring.perCall())` |
| `.monitored(PortMonitoring.perCall().businessExceptions(A.class, B.class))` | the same, with `A` and `B` (and their subtypes) counted as answers |
| `.monitored(PortMonitoring.summarized(Duration.ofMinutes(1)))` | one summary per method and caller per minute |
| `.monitored(PortMonitoring.perCall().summarizing(Duration.ofMinutes(1), "isMember"))` | `isMember` summarized, every other method per call |

The binding checks what it can when it is made:

- **Only an interface can be monitored**, since the proxy implements it:
  ```
  IllegalArgumentException: port '...RecordingRoutes' cannot be monitored: only an interface can be,
  since the calls are intercepted by a java.lang.reflect.Proxy. Bind it without monitored(), or put an
  interface in front of it
  ```
- **A method named in `summarizing(...)` must exist** on the port: `port '...CustomerDirectory' has no method 'lookup' to summarize`.
- **A summary interval is at least one second.**
- **A `PortUnavailableException` cannot be declared a business exception** — nor can a supertype of it such as `RuntimeException`, which would swallow it too. Counting an outage as an answer would stop it from ever counting as a failure, so `businessExceptions(...)` refuses it with an `IllegalArgumentException`.

What the caller receives does not change: the proxy throws exactly what the adapter threw — the `UnknownCustomerException` itself, not a wrapper — and the call's return value is the adapter's.

### Three Outcomes, Three Events

Every monitored call ends as one `BoundedContextEvent`, delivered to the context's listeners like every other kernel event — to a log with `LoggingBoundedContextListener`, to a monitoring stream with `StreamAppendingBoundedContextListener` (see [Observability and Tracing](/posts/eventmodeling-observability/)):

| event | when | carries |
|---|---|---|
| `PortCalled` | the call returned | port, qualification, method, caller, slice, `durationMicros` |
| `PortCallRejected` | it threw a business exception | the same, plus `exceptionType` and `reason` (the message) — **no stack trace** |
| `PortCallFailed` | it threw anything else | the same, plus the whole `Failure`, stack trace included |

This is the split commands make between `CommandRejected` and `CommandFailed`, one layer out, and for the same reason. A declined transfer is the gateway doing its job; if it were counted as a failure, a busy afternoon of declined cards would make the gateway look unhealthy and page someone. Alert on `PortCallFailed`; chart `PortCallRejected` by reason. The two layers agree by construction: a command that lets a port's business exception through is reported `CommandRejected`, a failure `CommandFailed`.

`durationMicros` is how long the caller was blocked in the call, measured before anything is reported. Reporting is contained: a monitoring store that fails never fails the port call.

What one call through the customer directory looks like:

```
PortCalled[boundedContext=banking, port=CustomerDirectory, qualification=null, method=verify,
           caller=PortCaller[kind=SLICE, name=OpenAccount], durationMicros=59,
           slice=FeatureSlice[name=OpenAccount, type=STATE_CHANGE, ...]]

PortCallRejected[boundedContext=banking, port=CustomerDirectory, qualification=null, method=verify,
           caller=PortCaller[kind=SLICE, name=OpenAccount], durationMicros=121,
           exceptionType=com.example.banking.ports.CustomerDirectory$UnknownCustomerException,
           reason=unknown customer mallory, slice=FeatureSlice[name=OpenAccount, ...]]
```

### Summarized Ports

Per call, every port call is one more append to the monitoring store, on the caller's thread. That is right for a gateway called a few times a minute and wrong for a lookup made on every request or for every projected event: the monitoring traffic would match the port's.

A summarized port condenses its calls into one **`PortCallsSummarized`** per port, method and caller per interval:

```
PortCallsSummarized[boundedContext=banking, port=MessageBus, qualification=null, method=send,
    caller=PortCaller[kind=DISPATCHER, name=AccountAnnouncedDispatcher],
    windowStart=2026-10-12T09:48:00Z, windowEnd=2026-10-12T09:48:11.519253991Z,
    called=1, rejected=0, failed=0, totalMicros=1194, maxMicros=1194,
    latencyBuckets=[0, 0, 0, 0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0], slice=FeatureSlice[name=AnnounceAccount, ...]]
```

- **Windows are aligned to the wall clock**, so the windows of several instances of one deployment cover the same span and add up.
- A summary is emitted at the end of each window that had calls, and **for the unfinished window when the context stops or terminates** — the example above was cut short by a `terminate()` eleven seconds in.
- **A failure is still emitted on its own**: the first of each exception type per method and caller per window, as a `PortCallFailed` with `summarized = true` (so a reader adding up calls does not count it twice). A failure is the one outcome somebody opens to read; the rest are counted.
- `summarizing(interval, "method"...)` does this for the hot methods of an otherwise per-call port — the one lookup made on every request, on a port whose other methods are rare.

### Latency Buckets

`latencyBuckets` counts the window's calls per fixed bucket, all outcomes together. The buckets are `PortLatencyBuckets.UPPER_BOUNDS_MICROS` — inclusive upper bounds from 100 µs to 10 s — plus one overflow bucket, `PortLatencyBuckets.COUNT` (17) in all:

```
100µs  250µs  500µs  1ms  2.5ms  5ms  10ms  25ms  50ms  100ms  250ms  500ms  1s  2.5s  5s  10s  overflow
```

They are fixed, and shared by everything that buckets a duration, because that is what makes histograms **mergeable**: windows of one instance, instances of one deployment, and per-call durations a dashboard buckets itself all add up bucket by bucket. Percentiles do not add up; buckets do. `PortLatencyBuckets` has the helpers:

```java
List<Long> merged = PortLatencyBuckets.merge(instanceA.latencyBuckets(), instanceB.latencyBuckets());
long p95 = PortLatencyBuckets.percentileMicros(merged, 95);   // the upper bound of the bucket it falls in
```

`percentileMicros` answers an upper estimate — the safe direction for a latency — `-1` for an empty histogram and `Long.MAX_VALUE` when the percentile falls in the overflow bucket.

## Who Called

Every port event names its **caller**, a `PortCaller(kind, name)`, and the caller's feature **slice**. A port is bound on the builder and belongs to no slice, so the slice on the event is always the slice of the code that made the call.

The framework knows who is running at every point it hands control to your code:

| `PortCaller.kind` | the call was made from | name |
|---|---|---|
| `COMMAND` | a command's `execute` (or its evaluation) | the command's name |
| `AUTOMATION` | an automation's `handle` or `onFailure` | the automation's class name |
| `TRANSLATOR` | a translator | its class name |
| `DISPATCHER` | a dispatcher's `when` | its class name |
| `READ_MODEL` | a read model's `when`, or a live read | its `readmodelName()` |
| `SLICE` | code a slice wired in its `start...` hooks, outside any component | the slice's name |
| `UNATTRIBUTED` | a port taken from the built context by application code | — |

Scopes nest: a live read inside a command is the read model's for its duration, and the command's again after it.

**The `SLICE` row is what makes endpoints visible.** An endpoint runs on a request thread the framework did not start, where no component is running — which is exactly where a membership or permission check is made before a command is executed. Each slice's `start...` hooks are handed a view of the context whose `port(...)` reports calls as that slice's whenever no component is running on the thread. So the endpoint above shows up as `PortCaller[kind=SLICE, name=OpenAccount]`, whether it took the port at start or looks it up per request — and a command it executes through the same reference is still reported as the command.

The calls of the transfer automation, meanwhile, read `PortCaller[kind=AUTOMATION, name=ExecuteTransferAutomation]` with the slice `ExecuteTransfer`.

> A port taken from the built context by the application — `banking.port(MessageBus.class, "eu").send(...)` in a `main` — is `PortCaller.UNATTRIBUTED`, with no slice. That is accurate: no slice made the call. Wire calls through a slice's `start...` hook when you want them attributed.
{: .prompt-tip }

## One Request, One Flow: `TracingScope`

An endpoint that checks the customer directory and then executes a command does two things for one request. Left alone, they would be two unrelated flows in the monitoring record — the port call with a correlation id of its own, the command with another. The edge prevents that by binding the request's tracing to the thread once, where the request enters the application:

```java
/** What an HTTP filter does around every request: one request, one flow. */
public static Object handle ( String actor, String correlationIdHeader, Supplier<Object> request ) {
    Tracing tracing = Tracing.actorAndChannel(actor, "api");          // mints a correlation id
    if ( correlationIdHeader != null ) {
        tracing = tracing.correlationId(correlationIdHeader);           // or continues the caller's
    }
    try ( TracingScope scope = TracingScope.bind(tracing) ) {
        return request.get();
    }
}
```

Two things pick the bound tracing up:

- **A monitored port call with no component tracing** — the endpoint's own `directory.verify(...)` — is reported under it, so its `PortCalled` carries `x-correlation-id` of the request.
- **The no-tracing overloads of the context's capabilities** — `execute(command)`, `read(...)`, `evaluate(...)` — use it instead of minting a fresh one, so the `AccountOpened` the command raises carries the same `x-correlation-id`, `x-actor:alice` and `x-channel:api`.

Called as `handle("alice", "req-42", ...)`, the port event and the `AccountOpened` event both carry `x-correlation-id:req-42`. The publisher's `AccountAnnounced` continues the same flow, since a publisher carries the correlation id of the event it publishes.

The rules of the binding:

- An **explicit tracing always wins** over the bound one, and a component running on the thread keeps its own.
- A binding **belongs to its thread**. Nothing is inherited by a thread the request starts, nor by the processor threads of automations and dispatchers, which carry the tracing of the events they handle.
- `close()` restores what was bound before, so scopes nest; it is idempotent and does nothing on another thread. `TracingScope.bind(null)` is an `IllegalArgumentException` — an edge that forgot its tracing should be visible.

## The Inventory: `BoundedContextStarting.ports`

Before a single call is made, a starting context announces every binding in its `BoundedContextStarting` event — monitored or not — as a list of `PortBinding`s. That is what a deployment depends on, readable from the monitoring stream:

```
PortBinding[port=HttpRoutes, portType=com.example.banking.HttpRoutes, qualification=null,
            adapter=...RecordingRoutes, monitored=false, monitoring=null, summarizedMethods=[], slices=[]]
PortBinding[port=CustomerDirectory, ..., monitored=true, monitoring=per call, summarizedMethods=[], slices=[]]
PortBinding[port=MessageBus, ..., monitored=true, monitoring=summarized every PT1M,
            summarizedMethods=[*], slices=[AnnounceAccount]]
PortBinding[port=TransferGateway, ..., monitored=true, monitoring=per call, summarizedMethods=[],
            slices=[ExecuteTransfer]]
```

`slices` lists the slices that asked for the port **while being configured**. A port taken in a `start...` hook is not listed there — the inventory is announced before any slice starts — but every call through it names its slice on the event itself. `summarizedMethods` is `[*]` for a fully summarized port.

## The Observer Sees Every Call

Independently of the events, a context's `BoundedContextObserver` is told of every call through a monitored port as an `Observation.PortCall`, **whatever the binding's mode** — summarizing decides what reaches the monitoring stream, never what the observer sees. It completes with `Outcome.PortReturned` or `Outcome.PortRejected` (type and reason), or fails with the throwable.

The observation is started on the caller's thread before the adapter runs, so it nests under whatever made the call: the transfer gateway's calls arrive nested in the automation's `AutomationRun`, a command's calls in its `CommandExecution`. A tracing binding gets the port as a child span with nothing to propagate. See [Observability and Tracing](/posts/eventmodeling-observability/).

## When Not to Monitor

A port is not only where a dependency is bound; it is also where configuration and infrastructure reach a slice. Those are not calls worth reporting:

| binding | monitor? | why |
|---|---|---|
| an external system: gateway, register, mail server, bus | **yes** | its calls are the ones that fail, slow down and decline |
| a hot lookup made per request or per event | **yes, summarized** | per-call reporting would double the load on the monitoring store |
| the application's router (`HttpRoutes`) | no | registering a route at start is not a call anyone needs timed |
| a `DataSource` | no | the proxy would time `getConnection()` — pool checkout, not the query |
| a `String` path, a config record | no | not an interface, and nothing to time |

Monitoring is opt-in per binding for exactly this reason. Leave configuration bindings plain: `.adapter(routes).forPort(HttpRoutes.class)`.

## Testing with Ports

Nothing in a port needs the framework to test. The components that use one take it in their constructor, so a test hands them a fake directly:

```java
/** A gateway the test scripts: each call takes the next answer. */
private final Deque<Supplier<String>> answers = new ArrayDeque<>();
private final TransferGateway gateway = ( toIban, amountInCents, key ) -> answers.pop().get();

@Override
public Automation<TransferToExecute, BankingDomainEvent, BankingOutboundEvent> automation ( ) {
    return new ExecuteTransferAutomation(new TransfersToExecuteTodoList(), gateway);
}

@Test
void anUnreachableGatewayKeepsTheItem ( ) {
    answers.add(() -> { throw new GatewayUnavailableException("connection refused"); });
    answers.add(() -> "gw-1");
    given()
        .event(new TransferRequested(T_1, ACC_1, "BE01", 100), TRANSFER.tags(T_1))
        .whenBatchRuns()
            .itemsFailed(1)
            .noEvents()
        .and()
        .whenBatchRuns()
            .itemsHandled(1)
            .events(new TransferExecuted(T_1, "gw-1"));
}
```

To assert on the monitoring itself — that a decline is a `PortCallRejected`, that an endpoint's call is the slice's — build a context with a listener that collects what it is told, bind a fake adapter `.monitored(...)`, and inspect the events. [Testing Feature Slices](/posts/eventmodeling-testing/) covers the test bases.

## Summary

| you want | write |
|---|---|
| a dependency outside the context | an interface in the slice's package, or in a shared package for several slices |
| the adapter bound | `.adapter(x).forPort(Port.class)`, or `forPort(Port.class, "qualification")` |
| the port in a component | `builder.port(Port.class)` in `configure...` |
| the port in an endpoint | `context.port(Port.class)` in `start...` |
| every call reported | `.monitored()` |
| a third-party "no" counted as an answer | `PortMonitoring.perCall().businessExceptions(TheirException.class)` |
| an outage counted as a failure, always | an exception extending `PortUnavailableException` |
| a busy port reported cheaply | `PortMonitoring.summarized(interval)` or `.summarizing(interval, "method")` |
| one request as one flow | `TracingScope.bind(tracing)` at the edge |
| what a deployment depends on | `BoundedContextStarting.ports` |
