---
layout: post
toc: true
title: Where a Validation Goes
description: One place for each kind of check, and why a validation runs before events exist or never
date: 2026-10-11 04:00:00
categories: [Eventmodeling Documentation,Eventmodeling Guides]
tags: [eventmodeling, validation, commands, decision models, dcb, uniqueness, poison events]
---

Every validation answers one of three questions: *can this data be said at all* (structure), *is this
request well-formed* (input), and *does history permit it* (invariants). The framework has one place for
each, and most validation bugs are a check written in the wrong one — usually a check that runs against
events that already exist.

That is the rule this guide hangs on: **a validation runs before events exist, or never.** The write path
validates; the read path accepts. An event that was appended is a fact, and code that reads facts — a
payload record's constructor, an upcaster, a projection, a translator — is not being asked for an
opinion. A rule enforced on the read path does not reject bad data, because the data is already there; it
rejects *your ability to read it*, which is strictly worse.

Each step below runs later than the one before it and guards something the earlier ones cannot. It is the
sibling of [Choosing a Read Model](/posts/eventmodeling-read-models/), and step 3 is extended by
[Business Rules and Enforcement Levels](/posts/eventmodeling-business-rules/).

## 1. Value Objects: Validity That Travels with the Data

```java
// carried inside an event payload: a lenient canonical constructor, validation in the factory
record EmailAddress ( String value ) {
    public static EmailAddress of ( String raw ) {
        if ( raw == null || !raw.contains("@") ) {
            throw new IllegalArgumentException("not an email address: " + raw);
        }
        return new EmailAddress(raw.strip().toLowerCase());
    }
}

// never persisted (a read model's answer): validate as hard as you like
public record AccountDetails ( AccountId accountId, CustomerId customerId, LocalDate openDate ) {
    public AccountDetails {
        if ( accountId == null || customerId == null || openDate == null ) {
            throw new IllegalArgumentException("all fields are required");
        }
    }
}
```

A record with a validating static factory gives you what you want from a value object — an instance
implies the rules held — and the factory is where *normalisation* belongs too (strip, lowercase), so that
equality and tag matching see one spelling. The entities follow the same split: an `EntityId` record is a
plain carrier, and `ACCOUNT.id(raw)` strips the value and rejects a blank one, so the rule lives in one
place per kind of entity.

**Where the record is carried inside an event payload, the canonical constructor must stay lenient.**
Events are deserialized *through the canonical constructor* on every read of history. A compact
constructor that throws turns a tightened rule into a poison event: the first event written before the
rule that fails it becomes an `EventDeserializationException` — never retryable, raised by whatever read
touches it, and through a projector it stops the read model. Nothing fails when you add the rule; it fails
months later, on data that was valid when it was written. For a record nothing persists — a read model's
answer, a command's parameter — a strict compact constructor is fine and better.

The cost of the lenient constructor is that an instance only implies validity when it came from the
factory. The published `AbstractBoundedContextArchUnitTest` closes that gap: among its checks, a record
exposing a static factory must not have its constructor called from anywhere else. Extend it with an empty
class in your root package:

```java
public class BankingArchUnitTest extends AbstractBoundedContextArchUnitTest { }
```

## 2. Reject the Malformed Request, Before Any History Is Read

```java
public record DepositCommand ( AccountId accountId, BigDecimal amount, String description, Overrides overrides )
        implements Command<BankingDomainEvent>, Overriding {

    public DepositCommand {
        if ( accountId == null || amount == null || amount.signum() <= 0 ) {
            throw new IllegalArgumentException("a deposit needs an account and a positive amount");
        }
        overrides = ( overrides == null ) ? Overrides.none() : overrides;
    }
    ...
}
```

Input validation is about the request, not the world: no event history can make a null account id
acceptable. It belongs in the command's constructor (or at the top of `execute`, before
`decisionModels(...)`), so a malformed request costs nothing — no projection, no append, no failure
anyone needs to alert on. `IllegalArgumentException` is the right type: the caller sent something
unusable, and the identical request can never succeed.

A command is not a payload record — it is never stored — so its compact constructor may throw. And when a
parameter is a value object, step 1 already did the work: a command taking `EmailAddress` cannot receive a
malformed email.

## 3. Rules on History: Decide on a Decision Model, Let the Append Re-check

```java
@Override
public void execute ( CommandContext<BankingDomainEvent, BankingDomainEvent> context ) {

    var account = new AccountBalanceDecisionModel(accountId);
    var exceptionsToday = new OverdraftExceptionsTodayDecisionModel(context.actor(), today);
    var result = context.decisionModels(account, exceptionsToday);   // projects AND pins the boundary

    // requests that make no sense: rejected, there is nothing to override
    BusinessException.when(!account.accountExists(), "Account does not exist");
    BusinessException.when(account.isClosed(), "Account is closed");

    // the bank's rules, each at its own enforcement level
    BigDecimal balanceAfter = account.balance().subtract(amount);

    context.check(MAXIMUM_WITHDRAWAL, amount.compareTo(MAXIMUM_WITHDRAWAL_AMOUNT) > 0,
        "A withdrawal of " + amount + " exceeds the maximum of " + MAXIMUM_WITHDRAWAL_AMOUNT);

    context.check(NO_OVERDRAFT, balanceAfter.signum() < 0, "The balance would become " + balanceAfter)
        .overridableWhen(exceptionsToday.mayGrantAnother(), exceptionsToday.whyNot());
    ...

    result.raiseEvent(new MoneyWithdrawn(accountId, amount, description, context.ruleViolations()),
        ACCOUNT.tags(accountId));
}
```

This is where invariants live, and the shape is always the same: **select decision models, check,
raise.** A `DecisionModel` is a projection the command builds inline — `eventQuery()` scoped by tags to
the entity, `when(...)` folding state, accessors for the checks — see
[Commands and Decision Models](/posts/eventmodeling-commands-and-decision-models/).

There are two kinds of "no" here, and they get different tools:

- **A request that makes no sense** — the account does not exist, it is closed — is a
  `BusinessException`, thrown on the spot. No override can make it meaningful.
- **A rule people may break under conditions** — an overdraft a teller may grant, advice that never
  blocks — is a business rule with an enforcement level, checked with `context.check(...)` and judged by
  the kernel after the command ran. It has its [own guide](/posts/eventmodeling-business-rules/).

What makes this more than a lookup: `context.decisionModels(...)` puts the events the models were read
from **inside the command's consistency boundary**. The lock filter of the append is the union of every
query the models were read with — each `eventQuery()`, and a savepoint model's `initQuery()` — so the
check at append time re-asks exactly the question the command decided on. Three different failures come
out of this step, and they want different responses:

| Exception | Thrown by | Means | Retry? |
|---|---|---|---|
| `IllegalArgumentException` | the command, before any read | the request is malformed | never |
| `BusinessException` | the command, after projecting | history says no | never — the answer *is* no |
| `OptimisticLockingException` | the append | new relevant facts since the decision | yes — re-execute: `executeWithRetry` |

The kernel also reports each outcome as its own event — `CommandRejected` (the reason, nothing else),
`CommandFailedOnOptimisticLocking`, and `CommandFailed` (type, message and stack trace) — so a rejection
is visible without being anybody's failure, and a dashboard can alert on failures while charting
rejections.

> **A rule rejection is a `BusinessException`, never an `IllegalStateException`.** That is what lets a
> catch block and the monitoring record tell a rejection from a bug. An `IllegalStateException` is
> reported as `CommandFailed`, with a stack trace, because the kernel cannot tell it from one.
{: .prompt-warning }

**A rule that needs no history needs no decision model.** `noDecisionModels()` is the declaration that
this command's append needs no guard, and hands the command its `CommandResult`.

**Bound the replay, never the boundary.** A decision model is replayed on every execution, so bound it
the way a live read model is bounded: by tags, and with a savepoint `initQuery()`. An `until` on a model's
own query bounds its read only — the framework strips it from the lock filter, because a filter carrying
one would admit every append.

## 4. Uniqueness: the Boundary You Expect to Be Empty

```java
public class EmailNotTakenDecisionModel implements DecisionModel<BankingDomainEvent> {

    private final EmailAddress email;
    private boolean taken;

    public EmailNotTakenDecisionModel ( EmailAddress email ) {
        this.email = email;
    }

    @Override
    public EventQuery eventQuery ( ) {
        return EventQuery.forEvents(EventTypesFilter.of(CustomerRegistered.class), Tags.of("email", email.value()));
    }

    @Override
    public void when ( Event<BankingDomainEvent> event ) {
        taken = true;
    }

    public boolean taken ( ) {
        return taken;
    }
}

// in RegisterCustomerCommand.execute:
var claim = new EmailNotTakenDecisionModel(email);
var result = context.decisionModels(claim);

BusinessException.when(claim.taken(), "Email address already registered: " + email.value());

CustomerId customerId = CUSTOMER.newId();
result.raiseEvent(new CustomerRegistered(customerId, email),
    Tags.of(CUSTOMER.tag(customerId), Tag.of("email", email.value())));
```

"No two customers with one email address" is a rule about a *set*, not an entity, and the reflex is to
check a read model first — which is a race (see the shapes to avoid below). The event-sourced answer is a
consistency boundary you expect to be empty: a decision model whose query matches the claim, reading
**nothing**. An empty read is still a boundary — the head is pinned before the read, and the append is
admitted only if *still* nothing matches. Of two concurrent registrations of the same address exactly one
wins; the other gets an `OptimisticLockingException`, and under `executeWithRetry` its re-decide turns
that into the `BusinessException`. The `taken()` check gives the ordinary sequential case its message;
the boundary is what makes the concurrent case safe.

Two things make the tag reliable as the uniqueness key:

- **Matching is exact** — no prefix, no wildcard, no case folding — so the value must be normalised to one
  spelling before it becomes a tag. That is step 1's job: `EmailAddress.of` lowercases and strips.
- **The claim must be on the event *and* in the query.** An event raised without the tag is invisible to
  the boundary that is supposed to defend it. Assert the tag in the command's test.

The claim events stay forever; releasing a claim is another event the model folds. And the boundary
serialises concurrent registrations of one address — which is exactly what you asked for.

## 5. The Edges: Accept What Arrived, Validate What You Do About It

Everything above runs inside a command, before the append. At the edges of the context the posture
inverts: **what has arrived can only be accepted; what you do about it is where the validation goes.**

- **A translator must not throw on an inbound event.** Translators run behind a projector. A throw is
  retried with backoff, and the same event throws again on every retry: that translator stalls on it,
  every inbound event behind it waiting, reported as `TranslatorFailed` each round. Decide about an
  invalid inbound event in-band instead — ignore it (a `default -> { }` arm), or record the rejection as a
  domain event, which makes the dead-letter view an ordinary read model. Through the synchronous
  `translate(...)` path an exception propagates to the caller, and there it may validate loudly, because
  there is a caller to reject.
- **An automation classifies the failure of its effect in `onFailure`.** The external call it makes can
  fail in ways only the automation can judge, and `AutomationFailureAction` is that judgement: retry the
  item, let independent work overtake it, or stop for a human. The payments example's
  [`ExecutePaymentAutomation`](https://github.com/sliceworkz/eventmodeling/blob/develop/sliceworkz-eventmodeling-examples/src/main/java/org/sliceworkz/eventmodeling/examples/payments/features/executepayment/ExecutePaymentAutomation.java)
  maps each failure to an action *and* an event. See [Automations and Policies](/posts/eventmodeling-automations-and-policies/).
- **A policy validates nothing, and says up front what its command's "no" means.** It maps a domain event
  onto a command, and the command validates as it would for a user. Whether a rejection is an answer
  (`skipRejections()`: record it and move on) or a sign that something is wrong (`stallOnRejection()`:
  hold everything behind it until someone looks) is chosen when the policy is registered — there is no
  default, because only the domain knows which it is.
- **A duplicate submission is an idempotency key, not a check.** "This request was already processed"
  cannot be validated by reading first: the duplicate can land between the read and the append. Execute
  with a key derived from the request — never from the attempt — and the store ignores the repeat; an
  empty result means the work was already done, which for an at-least-once caller is success.
- **A publisher validates nothing.** It maps a fact a command already decided on into the published
  language. Whether to publish at all is a command's decision, recorded as a domain event the publisher
  then maps.

## 6. Evolving a Rule Without Breaking History

A new or tightened rule changes what may be *written from now on*. The events already stored were valid
under the rules of their day, and they stay readable under yours.

- **The rule lands in the factory (step 1) and the command (steps 2–4).** Never in the canonical
  constructor of a persisted record, and never in an upcaster: an upcaster runs on the read path, and one
  that throws on old data manufactures the same poison event a throwing constructor does.
- **Where old data must be brought to the new shape, an upcaster *converts*** — defaults a missing field,
  maps a retired value. It does not *judge*.
- **If stored events genuinely violate a new invariant, that is a domain fact**, not a serialization
  problem: record a correcting event, or have the decision model treat the old state explicitly. The store
  offers no way to reject history after the fact, on purpose.

## Shapes to Avoid

**A throwing compact constructor on a record an event carries.** Nothing fails at the time; the rule
waits for the first read of an event written before it. Step 1 shows where the same rule goes safely.

**Deciding on a read model.** `context.read(...)` is outside the consistency boundary. "Check first, then
execute" against a read model is a race: the append succeeds against history the check never saw.
Anything a raised event depends on belongs in a `DecisionModel`.

**Validating in a projection's `when()` — a read model's or a translator's.** The read path again, with a
wider blast radius. The projector retries a failing projection with backoff, but an event the projection
refuses is refused on every retry: the read model or translator stays stuck on it, everything behind it
waiting, a `...Failed` event per round, until the code changes. A projection that meets an event it
considers impossible records or skips it; it does not throw at it.

**A uniqueness check without the boundary.** `claim.taken()` checked against anything but a decision
model is the read-model race again — correct in every test, wrong under concurrency. The empty boundary
(step 4) is what holds; the sequential check on top of it is just the better error message.

**An unkeyed effect on an at-least-once path.** An automation retries whatever did not complete. A
command it executes without a key derived from the todo item records the same fact twice, on exactly the
retries the design promises will happen.

**Rules in an upcaster.** Converting is its job; judging produces poison events.

## Quick Reference

| | Runs | Guards against | Surfaces as | Retry? |
|---|---|---|---|---|
| Value object factory | at construction | data that cannot be said | exception at the edge | no — fix the input |
| Input check (command constructor) | before any read | a malformed request | `IllegalArgumentException` | no |
| Decision model check | after projecting history | a request history rejects | `BusinessException`, `CommandRejected` | no — the answer is no |
| Business rule with an enforcement level | judged after the command ran | a rule people may break under conditions | `RuleViolationException` (a `BusinessException`) unless overridden, or recorded on the events | no — override, or change the request |
| DCB append check | inside the append | facts newer than the decision | `OptimisticLockingException` | yes — `executeWithRetry` |
| Empty boundary (uniqueness) | inside the append | a concurrent duplicate claim | `OptimisticLockingException`, then the re-decide's `BusinessException` | re-execute |
| Idempotency key | inside the append | the same request twice | an empty result, silently | no — already done |
| Automation `onFailure` | after an effect failed | a failing external effect | an `AutomationFailureAction` and an event | per the action |

## Testing the Rules

`CommandTest` speaks this guide's language:

```java
@Test
void aTakenEmailIsRejected ( ) {
    EmailAddress jane = EmailAddress.of("jane@example.com");
    given()
        .event(new CustomerRegistered(CUSTOMER.id("c-1"), jane), Tags.of("email", jane.value()))
        .when(new RegisterCustomerCommand(EmailAddress.of(" Jane@Example.com")))
        .then()
        .businessError("Email address already registered: jane@example.com");
}

@Test
void theClaimIsOnTheEvent ( ) {
    EmailAddress jane = EmailAddress.of("jane@example.com");
    given()
        .when(new RegisterCustomerCommand(jane))
        .then()
        .event(new CustomerRegistered(CUSTOMER.id(IGNORE_ID()), jane), Tags.of("email", "jane@example.com"));
}
```

- `businessError(message)` asserts the rejection is a `BusinessException`, so a rule thrown as the wrong
  type fails the test. `error(Class, message)` asserts any other type.
- `.event(expected, tags)` pins that the claim tag really is on the raised event; `.noEvents()` proves a
  rejection stored nothing.
- For business rules with an enforcement level, `as(actor)`, `whenEvaluated(...)` and
  `then().rulesViolated()` assert what the kernel made of them — see
  [Business Rules and Enforcement Levels](/posts/eventmodeling-business-rules/).
- The concurrent half of a uniqueness check is the storage's promise, held per backend by the
  eventstore's compliance suite. Your test asserts the sequential rejection and the tag; run it
  `@ForEachBackend` (see [Testing Feature Slices](/posts/eventmodeling-testing/)) to have it hold on the
  storage you deploy on.
