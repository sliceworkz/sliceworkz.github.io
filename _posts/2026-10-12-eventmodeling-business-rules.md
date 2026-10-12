---
layout: post
toc: true
title: Business Rules and Enforcement Levels
description: Rules people may break under conditions — declared once, checked by the command, judged by the kernel, previewed before anything is submitted
date: 2026-10-12 05:00:00
categories: [Eventmodeling Documentation,Eventmodeling Guides]
tags: [eventmodeling, business rules, sbvr, overrides, commands, evaluation, http]
---

Some of what a command checks is not a yes-or-no. The bank does not want accounts to go negative, but a
teller may make an exception. A large deposit is fine once somebody explains where the money comes from. A
withdrawal should have a description, and nothing stops one without. And before a user submits anything,
the screen should already show what will work, what will not, and what they are allowed to override.

The framework models this on the **enforcement levels** of SBVR, the OMG's *Semantics of Business
Vocabulary and Business Rules*. This guide covers how a rule is declared, how a command checks it, who may
override it, what gets recorded, how a user sees it all before submitting, and how the rules enforced
*after the fact* are followed up. It extends step 3 of [Where a Validation Goes](/posts/eventmodeling-where-validations-go/).

The repository's banking example uses every level:
[`WithdrawCommand`](https://github.com/sliceworkz/eventmodeling/blob/develop/sliceworkz-eventmodeling-examples/src/main/java/org/sliceworkz/eventmodeling/examples/banking/features/withdraw/WithdrawCommand.java),
[`DepositCommand`](https://github.com/sliceworkz/eventmodeling/blob/develop/sliceworkz-eventmodeling-examples/src/main/java/org/sliceworkz/eventmodeling/examples/banking/features/deposit/DepositCommand.java),
and [`BankingBusinessRulesExample`](https://github.com/sliceworkz/eventmodeling/blob/develop/sliceworkz-eventmodeling-examples/src/main/java/org/sliceworkz/eventmodeling/examples/banking/BankingBusinessRulesExample.java)
to run. The samples below are a simplified version of the same. Compared to the [quickstart](/posts/eventmodeling-quickstart/), the money movements here carry a description and the rule violations they went ahead with: `MoneyWithdrawn(AccountId accountId, BigDecimal amount, String description, List<RuleViolation> ruleViolations)`, and `MoneyDeposited` likewise.

## Two Kinds of "No"

SBVR separates two kinds of rule, and the framework has one tool for each:

| | SBVR calls it | Example | Breaking it | In a command |
|---|---|---|---|---|
| **The request makes no sense** | a *definitional* rule | "this account does not exist" | impossible: nothing to override | `BusinessException.when(...)`, thrown on the spot |
| **The business says you ought not** | a *behavioral* rule | "a withdrawal must not make the balance negative" | possible: people can do otherwise | `context.check(rule, violated, message)`, judged by its enforcement level |

Keep the first kind a `BusinessException`. No override can make a withdrawal from a non-existing account
meaningful, and reporting it as a rule would put a decision in front of the user where there is none.
Everything below is about the second kind.

## Declaring a Rule

```java
public interface BankingDomain {
    ...
    BigDecimal MAXIMUM_WITHDRAWAL_AMOUNT = new BigDecimal("50000");
    BigDecimal LARGE_WITHDRAWAL_AMOUNT   = new BigDecimal("5000");
    int DAILY_OVERDRAFT_EXCEPTIONS = 3;

    BusinessRule MAXIMUM_WITHDRAWAL = BusinessRule.of("maximum-withdrawal",
            "A single withdrawal must not exceed " + MAXIMUM_WITHDRAWAL_AMOUNT + ".")
        .enforcedAt(EnforcementLevel.STRICTLY_ENFORCED);

    BusinessRule NO_OVERDRAFT = BusinessRule.of("no-overdraft",
            "A withdrawal must not make the balance of the account negative.")
        .enforcedAt(EnforcementLevel.PRE_AUTHORIZED_OVERRIDE);

    BusinessRule LARGE_WITHDRAWAL_JUSTIFIED = BusinessRule.of("large-withdrawal-justified",
            "A withdrawal above " + LARGE_WITHDRAWAL_AMOUNT + " must be justified.")
        .enforcedAt(EnforcementLevel.POST_JUSTIFIED_OVERRIDE);

    BusinessRule WITHDRAWAL_DESCRIBED = BusinessRule.of("withdrawal-described",
            "A withdrawal should say what it is for.")
        .enforcedAt(EnforcementLevel.GUIDELINE);
    ...
}
```

A `BusinessRule` is an id, a statement and a level, and each has its own job:

- **The id is wire format.** A front end sends it back to override the rule, every recorded violation
  stores it, and it is the value of the rule tags on the events. One token, no whitespace (the record
  refuses whitespace), and never renamed casually — treat it like an event type name.
- **The statement is wording**: what the user is shown, in the language of the business. It is not
  stored, so it can be reworded freely.
- **The level is policy**, kept out of the rule's identity: two rules with the same id are the same rule,
  whatever their levels. "From next month overdrafts are strictly enforced" is one word in the declaration
  and nothing else. `BusinessRule.of(id, statement)` is strictly enforced — a rule nobody decided to relax
  is enforced.

Keep the rules with the rest of the domain's vocabulary, in the domain interface: the rulebook is part of
the ubiquitous language, and a rule one slice checks is often followed up by another.

**Declare them on the context** from the slice that checks them. Checking a rule does not need it; being
seen does — the declared rulebook is announced when the context starts (on `BoundedContextStarting`),
which is how a dashboard shows every rule with its statement and level, also the ones nobody has violated:

```java
@FeatureSlice(chapter = "Transactions")
public class WithdrawFeatureSlice implements Slice<Banking> {

    @Override
    public void configureCommand ( BoundedContextBuilder<Banking> builder ) {
        builder.command(WithdrawCommand.class);
        // the rulebook: announced when the context starts, statement and level included
        builder.businessRules(MAXIMUM_WITHDRAWAL, NO_OVERDRAFT, LARGE_WITHDRAWAL_JUSTIFIED, WITHDRAWAL_DESCRIBED);
    }
}
```

`businessRules(...)` accumulates over slices; two *different* declarations of one id fail the build.

## The Enforcement Levels

| Level | A violation… | A front end shows | Recorded on the events as |
|---|---|---|---|
| `STRICTLY_ENFORCED` | blocks, always | an error | — |
| `DEFERRED_ENFORCEMENT` | never blocks; enforced later | a notice | `ENFORCEMENT_DEFERRED` |
| `PRE_AUTHORIZED_OVERRIDE` | blocks, unless overridden by an actor **the command authorizes** | a checkbox — or an error saying why not | `OVERRIDDEN` |
| `POST_JUSTIFIED_OVERRIDE` | blocks, unless overridden; the override must be justified afterwards | a checkbox | `JUSTIFICATION_PENDING` |
| `OVERRIDE_WITH_EXPLANATION` | blocks, unless overridden **with an explanation** | a checkbox and a text field | `OVERRIDDEN`, explanation included |
| `GUIDELINE` | never blocks | a hint | `GUIDELINE_NOT_FOLLOWED` |

- **Strictly enforced** differs from a `BusinessException` in what it is, not in its outcome: a named
  behavioral rule, reported next to every other violation when a command is evaluated, and relaxable by
  changing its level rather than its code. *Banking: `maximum-withdrawal`.*
- **Pre-authorized override** is for actors who hold the authorization before the fact — and which
  actors those are is the command's decision. *Banking: `no-overdraft`, three exceptions per teller per day.*
- **Post-justified override** goes through on request and leaves an obligation behind. *Banking:
  `large-withdrawal-justified`, settled by `JustifyWithdrawalCommand`.*
- **Override with explanation** goes through when the actor explains why. *Banking:
  `origin-of-funds-explained` on large deposits.*
- **Deferred enforcement** is enforced, but not now: the command goes ahead and something enforces the
  rule later. *Banking: `balance-within-guarantee`.*
- **Guideline** is advice. *Banking: `withdrawal-described`.*

## Checking Rules in a Command

The shape is always the same: select decision models, reject what makes no sense, check the rules, raise.

```java
public record WithdrawCommand ( AccountId accountId, BigDecimal amount, String description, LocalDate today, Overrides overrides )
        implements Command<BankingDomainEvent>, Overriding {

    public WithdrawCommand {
        if ( accountId == null || amount == null || amount.signum() <= 0 || today == null ) {
            throw new IllegalArgumentException("a withdrawal needs an account, a positive amount and a day");
        }
        overrides = ( overrides == null ) ? Overrides.none() : overrides;
    }

    @Override
    public void execute ( CommandContext<BankingDomainEvent, BankingDomainEvent> context ) {

        var account = new AccountBalanceDecisionModel(accountId);
        var exceptionsToday = new OverdraftExceptionsTodayDecisionModel(context.actor(), today);
        var result = context.decisionModels(account, exceptionsToday);

        // requests that make no sense: rejected, there is nothing to override
        BusinessException.when(!account.accountExists(), "Account does not exist");
        BusinessException.when(account.isClosed(), "Account is closed");

        // the bank's rules, each at its own enforcement level
        BigDecimal balanceAfter = account.balance().subtract(amount);

        context.check(MAXIMUM_WITHDRAWAL, amount.compareTo(MAXIMUM_WITHDRAWAL_AMOUNT) > 0,
            "A withdrawal of " + amount + " exceeds the maximum of " + MAXIMUM_WITHDRAWAL_AMOUNT);

        context.check(NO_OVERDRAFT, balanceAfter.signum() < 0, "The balance would become " + balanceAfter)
            .overridableWhen(exceptionsToday.mayGrantAnother(), exceptionsToday.whyNot());

        context.check(LARGE_WITHDRAWAL_JUSTIFIED, amount.compareTo(LARGE_WITHDRAWAL_AMOUNT) > 0,
            "A withdrawal of " + amount + " is above " + LARGE_WITHDRAWAL_AMOUNT + " and has to be justified afterwards");

        context.check(WITHDRAWAL_DESCRIBED, description == null || description.isBlank(), "No description given");

        // raise as if everything were allowed: the kernel stops what may not go ahead
        result.raiseEvent(new MoneyWithdrawn(accountId, amount, description, context.ruleViolations()),
            ACCOUNT.tags(accountId));
    }
}
```

- **`check` never throws.** It records whether the rule is violated and what to tell the actor — a
  message specific to this violation, where the statement is general. The kernel judges every check once
  the command has returned. That is what lets a command report **every** violated rule at once instead of
  the first, and what lets the same command be evaluated without being executed.
- **Raise as if everything were allowed.** The command does not branch on its checks. If a violation
  stops the execution, the kernel rejects it with a `RuleViolationException` before anything is appended.
  If it goes ahead, the event carries exactly the violations it went ahead with.
- **Check on decision models.** A rule judged on facts outside the consistency boundary is judged on facts
  that can change underneath it, so the checks come after `decisionModels(...)`.
- **Check everything before reading `ruleViolations()`.** The events must record the judgement the
  execution is made on, so a `check` or `overridableWhen` after that read is refused with an
  `IllegalStateException` — a bug, reported as `CommandFailed`.
- **Only commands check rules.** `check` is on `CommandContext`. An automation, a policy, a translator or a
  publisher decides nothing of its own; whatever needs deciding, it has a command decide.

## Who May Override: the Command Decides

Whether *this* actor may override *this* violation is usually a business rule of its own, and a subtle
one: a role granted through events, a maximum number of exceptions per actor per day. That needs history,
and history is what a command has in its decision models:

```java
public class OverdraftExceptionsTodayDecisionModel implements DecisionModel<BankingDomainEvent> {

    private final Optional<String> teller;
    private final LocalDate today;
    private int exceptionsToday;

    public OverdraftExceptionsTodayDecisionModel ( Optional<String> teller, LocalDate today ) {
        this.teller = teller;
        this.today = today;
    }

    @Override
    public EventQuery eventQuery ( ) {
        // the kernel tags every overridden withdrawal with the rule and the actor: one tag query
        return teller
            .map(t -> EventQuery.forEvents(EventTypesFilter.of(MoneyWithdrawn.class), RuleTags.overriddenBy(t, NO_OVERDRAFT)))
            .orElse(EventQuery.matchNone());
    }

    @Override
    public void when ( Event<BankingDomainEvent> event ) {
        if ( event.timestamp().atZone(ZoneOffset.UTC).toLocalDate().equals(today) ) {
            exceptionsToday++;
        }
    }

    public boolean mayGrantAnother ( ) {
        return teller.isPresent() && exceptionsToday < DAILY_OVERDRAFT_EXCEPTIONS;
    }

    public String whyNot ( ) {
        return teller.isEmpty()
            ? "Only an identified teller can authorize an overdraft"
            : "You already granted " + exceptionsToday + " overdraft exceptions today, the maximum is " + DAILY_OVERDRAFT_EXCEPTIONS;
    }
}
```

The command passes the answer to `overridableWhen(authorized, whyNot)` on the check:

- **It sits inside the consistency boundary.** The quota model is one of the command's decision models,
  so its query is part of the lock filter. Two overdrafts by the same teller racing past the quota cannot
  both succeed: the second append conflicts, and `executeWithRetry` re-decides on a count that includes
  the first.
- **`whyNot` is what the actor is shown** instead of a checkbox when the answer is no.
- **The actor is `context.actor()`**, read from the tracing the command was executed with — the same value
  the kernel stores in the `x-actor` tag of every event. It is empty for an execution without an actor,
  and a decision that needs one should then say no.
- **Without `overridableWhen`**: nobody may make a pre-authorized override; anybody who asks may make a
  post-justified override or an override with explanation. On the other levels there is nothing to decide.

The history the quota counts is written by the kernel, not by the domain: every event raised with an
override carries `x-rule-overridden:<rule>`, and every event carries `x-actor`.
`RuleTags.overriddenBy(actor, rule)` is those two tags together. Nothing in `MoneyWithdrawn` had to be
designed for it.

## Overrides Are an Input of the Command

The overrides a user ticked are part of the request, like the amount, so they are a component of the
command. A command accepts them by implementing `Overriding` — for a record, a component
`Overrides overrides` is all it takes:

```java
new WithdrawCommand(account, new BigDecimal("250"), "kitchen", today, Overrides.of(NO_OVERDRAFT.id()));

new DepositCommand(account, new BigDecimal("20000"), "car sold",
    Overrides.none().with(ORIGIN_OF_FUNDS_EXPLAINED, "Proceeds of selling a car"));
```

- **A request is not a grant.** The kernel weighs it against the rule's level and, for a pre-authorized
  override, against the command's own decision.
- **A request for a rule that is not violated is ignored** and records nothing, so a front end may send
  back every box that is ticked.
- **A command that does not implement `Overriding` overrides nothing.** That is the safe default for
  everything that builds commands without a user in front of it — automations, policies, translators,
  tests. A retry re-executes the same command, so it carries the same overrides and re-judges them on the
  facts as they are then.

## What Gets Recorded

An execution that goes ahead with violations records them twice, and each record has its own job:

- **In the payload**, as `List<RuleViolation>` from `context.ruleViolations()`: the rule id, the level in
  force, the disposition, the command's message and the actor's explanation. This is the business fact
  ("this withdrawal was an exception to the overdraft rule"), and the only place free text such as an
  explanation belongs. Only violations that did not block are in it. `RuleViolation` is an ordinary
  payload record: its constructor is lenient, as every payload record's must be.
- **As tags**, added by the kernel to every event of the append, whether or not the command also put the
  violations in its payload:

| Disposition | Tags |
|---|---|
| `OVERRIDDEN` | `x-rule-overridden:<rule>` |
| `JUSTIFICATION_PENDING` | `x-rule-overridden:<rule>`, `x-rule-justification-pending:<rule>` |
| `ENFORCEMENT_DEFERRED` | `x-rule-deferred:<rule>` |
| `GUIDELINE_NOT_FOLLOWED` | `x-rule-not-followed:<rule>` |

The tags make violations queryable without knowing any payload: by a decision model counting an actor's
overrides, by a todo list collecting what is still to be justified, by an auditor. `RuleTags` has the
constants and the helpers (`RuleTags.overridden(rule)`, `RuleTags.deferred(rule)`, …).

**Who made the exception** is the `x-actor` tag the kernel writes on every event anyway — read it back
with `Tracing.readFrom(event).actor()`. It is deliberately not repeated in the payload. A domain that wants
an explicit event for an exception (`OverdraftApproved`) raises one in the same append.

## Evaluating Before Executing

```java
Evaluation anonymous = bank.evaluate(overdraft);           // nothing is appended
Evaluation teller    = bank.evaluate(overdraft, alice);
```

`evaluate` runs the command exactly as `execute` would — decision models projected, its
`BusinessException`s thrown, its rules checked and judged against the overrides it carries — and stops
short of the append. It answers an `Evaluation`:

| Outcome | Means |
|---|---|
| `REJECTED` | the command threw a `BusinessException`: the request makes no sense (`rejection()` says why) |
| `BLOCKED` | at least one violation blocks for this actor |
| `NEEDS_OVERRIDE` | nothing blocks, but at least one violation needs an override this actor may give |
| `WOULD_SUCCEED` | nothing stops it; there may still be advice, deferred enforcements and accepted overrides |

and a `RuleJudgement` per violated rule, in the order the command checked them. For the overdraft above
the anonymous caller is `BLOCKED` ("Only an identified teller can authorize an overdraft") and the teller
gets `NEEDS_OVERRIDE`. A judgement's `verdict` is what a front end lays out:

| Verdict | Render |
|---|---|
| `BLOCKS` | an error, with `reason()` when the level allows overrides but the command said no for this actor |
| `OVERRIDE_REQUIRED` | a checkbox; with a text field when `explanationRequired()` |
| `OVERRIDDEN` | a ticked checkbox |
| `DEFERRED` | a notice |
| `ADVISED` | a hint |

What evaluating deliberately does **not** do:

- **It appends nothing, spends no idempotency key and emits no `CommandExecuted` or `CommandRejected`.** A
  front end may evaluate on every change of a form, and the monitoring record is of what happened, not of
  what was previewed.
- **It promises nothing about the execution that follows.** The facts may change in between, and the
  execution judges every rule again. An evaluation is a preview, not a reservation.
- **It is on `ApplicationCapabilities`, beside `execute`.** The code that submits a command is the code
  that previews it. Automations and translators have no user to show a preview to, and their contexts do
  not carry it.

When an execution is rejected on its rules, the `RuleViolationException` carries the same `Evaluation`
(`e.evaluation()`). A front end therefore has **one shape to render**, whether it checked beforehand or
just submitted. The cheapest front end skips the preview: it submits, gets the violations back, shows the
checkboxes, and submits again.

## Binding to HTTP

Previewing and executing are two methods on one resource, carrying the same body. With
[Javalin](https://javalin.io/) 7:

```java
record WithdrawalRequest ( String accountId, BigDecimal amount, String description, List<OverrideRequest> overrides ) { }

static Javalin create ( BankingApi banking ) {
    return Javalin.create(config -> {
        config.routes.query("/api/withdrawals", ctx -> ctx
            .header("Cache-Control", "no-store")
            .json(banking.evaluate(toCommand(ctx), tracingOf(ctx))));

        config.routes.post("/api/withdrawals", ctx -> {
            banking.executeWithRetry(toCommand(ctx), tracingOf(ctx), RetryPolicy.DEFAULT);
            ctx.status(201);
        });

        config.routes.exception(RuleViolationException.class, (e, ctx) -> ctx.status(422).json(e.evaluation()));
        config.routes.exception(BusinessException.class, (e, ctx) -> ctx.status(400).result(e.getMessage()));
        config.routes.exception(OptimisticLockingException.class, (e, ctx) -> ctx.status(409));
    });
}

static WithdrawCommand toCommand ( Context ctx ) {
    WithdrawalRequest request = ctx.bodyAsClass(WithdrawalRequest.class);
    return new WithdrawCommand(ACCOUNT.id(request.accountId()), request.amount(), request.description(),
        LocalDate.now(ZoneOffset.UTC), Overrides.of(request.overrides()));
}

static Tracing tracingOf ( Context ctx ) {
    return Tracing.actorAndChannel(ctx.attribute("username"), "api");   // set by your authentication handler
}
```

| | Request | Response |
|---|---|---|
| preview | `QUERY /api/withdrawals` with the command's body | `200` with the evaluation, **also when it is blocked**: the preview itself succeeded |
| execute | `POST /api/withdrawals` with the same body | `201`; `422` with the evaluation when rejected on its rules; `409` on a conflict that survived the retries |

```json
{ "accountId": "acc-1", "amount": 250, "description": "Kitchen",
  "overrides": [ { "rule": "no-overdraft" },
                 { "rule": "large-withdrawal-justified", "explanation": "Customer is waiting" } ] }
```

The evaluation renders as its components, and each judgement also carries `overridable` and
`explanationRequired`, the two facts a front end lays its widgets out by. A `QUERY` for a teller
overdrawing by 150 answers:

```json
{ "outcome": "NEEDS_OVERRIDE", "rejection": null,
  "judgements": [ { "rule": "no-overdraft",
                    "statement": "A withdrawal must not make the balance of the account negative.",
                    "enforcementLevel": "PRE_AUTHORIZED_OVERRIDE", "message": "The balance would become -150",
                    "verdict": "OVERRIDE_REQUIRED", "reason": null, "explanation": null,
                    "overridable": true, "explanationRequired": false } ] }
```

- **`QUERY` is the HTTP method for a safe, idempotent request that carries a body**, which is exactly what
  an evaluation is. On the same URI as the `POST` it is one resource with two methods, rather than a
  second "check" resource to keep in step with the first.
- **`Cache-Control: no-store`.** `QUERY` responses may be cached, keyed on the body, but an evaluation
  depends on the current state and on who is asking.
- **Cross-origin callers need `QUERY` in the CORS configuration**: it is not a CORS-safelisted method, so
  it triggers a preflight. Check that proxies and gateways in between let it through; OpenAPI describes a
  `query` operation from 3.2.
- **Overrides go in the body**, because they are the user's decision, like every other field of the form.
- **The actor comes from your authentication**, never from a field of the body: it decides who may
  override.

## Following Up After the Fact

Two levels are enforced after the command: post-justified override and deferred enforcement. Neither needs
anything from the framework beyond the recorded violation — the follow-up is an ordinary slice, fed by the
tag the kernel wrote. The command that finishes a follow-up says which violation it settles, and the
kernel links the two:

```java
@Override
public void execute ( CommandContext<BankingDomainEvent, BankingDomainEvent> context ) {
    var state = new JustificationState(accountId, withdrawal);
    var result = context.decisionModels(state);

    BusinessException.when(!state.pending, "Withdrawal " + withdrawal + " needs no justification");
    BusinessException.when(state.justified, "Withdrawal " + withdrawal + " was already justified");

    result.raiseEvent(
        new WithdrawalJustified(accountId, withdrawal,
            context.justifies(LARGE_WITHDRAWAL_JUSTIFIED, EventId.of(withdrawal), justification.strip())),
        ACCOUNT.tags(accountId));
}
```

| Call | Tag on every event of the append | Returns |
|---|---|---|
| `context.justifies(rule, eventId, text)` | `x-rule-justified:<rule>@<event id>` | a `RuleFollowUp` of kind `JUSTIFIED` |
| `context.enforces(rule, eventId, note)` | `x-rule-enforced:<rule>@<event id>` | a `RuleFollowUp` of kind `ENFORCED` |

The pending tag stays on the original event for good, because events are never rewritten; the link tag
says it was settled, and the `x-actor` tag by whom — readable by anyone, without knowing the domain event
that did it. Record the returned `RuleFollowUp` in the payload: its text is free text and belongs there.
Whether the event needs the follow-up is the command's decision on its decision models; a follow-up in an
execution that raises nothing is refused with an `IllegalStateException`, since it would be recorded
nowhere.

In the banking example:

- **Post-justified override** (`features/justifywithdrawal`):
  `OverridesAwaitingJustificationReadModel` selects `MoneyWithdrawn` events by
  `x-rule-justification-pending:large-withdrawal-justified` and subtracts the `WithdrawalJustified`
  events; `JustifyWithdrawalCommand` settles one, as above. Escalating an override that is never justified
  is an automation over a todo list of the same shape.
- **Deferred enforcement** (`features/excessbalance`): `DepositsAboveGuaranteeTodoList` selects
  `MoneyDeposited` events by `x-rule-deferred:balance-within-guarantee`, and
  `ReportExcessBalanceAutomation` executes `ReportExcessBalanceCommand` for each, with an idempotency key
  derived from the deposit; the command links its report with `context.enforces(...)`.

```
DepositCommand → MoneyDeposited [x-rule-deferred] → DepositsAboveGuaranteeTodoList
               → ReportExcessBalanceAutomation → ReportExcessBalanceCommand → ExcessBalanceReported
```

**Auditing** needs no domain class at all: the rule tags and link tags on the events, the actor on every
event, the rulebook on `BoundedContextStarting`, and — for an execution the rules stopped — the judgements
on `CommandRejected.ruleJudgements`. The dashboard's Business Rules screen is built on exactly these.

## Testing

`CommandTest` speaks this guide's language:

```java
@Test
void aTellerIsOfferedTheOverdraftAsAnOverride ( ) {
    given()
        .event(new AccountOpened(ACCOUNT_1, CUSTOMER.id("cust-1"), TODAY), ACCOUNT.tags(ACCOUNT_1))
        .event(new MoneyDeposited(ACCOUNT_1, new BigDecimal("100"), "salary"), ACCOUNT.tags(ACCOUNT_1))
        .as("alice")
        .whenEvaluated(new WithdrawCommand(ACCOUNT_1, new BigDecimal("250"), "tv", TODAY, Overrides.none()))
        .thenEvaluation()
        .needsOverrideOf(NO_OVERDRAFT)
        .notViolated(WITHDRAWAL_DESCRIBED);
}

@Test
void aTellerGrantsAnOverdraftAndItIsRecorded ( ) {
    given()
        .event(new AccountOpened(ACCOUNT_1, CUSTOMER.id("cust-1"), TODAY), ACCOUNT.tags(ACCOUNT_1))
        .event(new MoneyDeposited(ACCOUNT_1, new BigDecimal("100"), "salary"), ACCOUNT.tags(ACCOUNT_1))
        .as("alice")
        .when(new WithdrawCommand(ACCOUNT_1, new BigDecimal("250"), "tv", TODAY, Overrides.of(NO_OVERDRAFT.id())))
        .then()
        .event(new MoneyWithdrawn(ACCOUNT_1, new BigDecimal("250"), "tv", List.of(
                new RuleViolation(NO_OVERDRAFT.id(), EnforcementLevel.PRE_AUTHORIZED_OVERRIDE, Disposition.OVERRIDDEN,
                    "The balance would become -150", null))),
            Tags.of(RuleTags.overridden(NO_OVERDRAFT)));
}

@Test
void theFourthOverdraftOfTheDayIsRefused ( ) {
    MoneyWithdrawn earlier = new MoneyWithdrawn(ACCOUNT_1, new BigDecimal("1"), "earlier", List.of(
        new RuleViolation(NO_OVERDRAFT.id(), EnforcementLevel.PRE_AUTHORIZED_OVERRIDE, Disposition.OVERRIDDEN, "earlier", null)));
    Tags overdraftTags = ACCOUNT.tags(ACCOUNT_1).merge(Tags.of(RuleTags.overridden(NO_OVERDRAFT)));
    given()
        .event(new AccountOpened(ACCOUNT_1, CUSTOMER.id("cust-1"), TODAY), ACCOUNT.tags(ACCOUNT_1))
        .as("alice")
        .event(earlier, overdraftTags)
        .event(earlier, overdraftTags)
        .event(earlier, overdraftTags)
        .when(new WithdrawCommand(ACCOUNT_1, new BigDecimal("10"), "coffee", TODAY, Overrides.of(NO_OVERDRAFT.id())))
        .then()
        .rulesViolated()
        .blockedBy(NO_OVERDRAFT)
        .notOverridable(NO_OVERDRAFT, "You already granted 3 overdraft exceptions today, the maximum is 3");
}
```

- **`as(actor)`** acts as that actor from that point on: events seeded after it carry the actor's
  `x-actor` tag, and the command is executed or evaluated for that actor — which is what `context.actor()`
  answers.
- **`whenEvaluated(...)`** evaluates instead of executing, and fails if anything was appended.
  **`thenEvaluation()`** and **`then().rulesViolated()`** both hand out the same assertions —
  `wouldSucceed()`, `needsOverrideOf(...)`, `blockedBy(...)`, `rejected(message)`, `judged(rule, verdict)`,
  `notOverridable(rule, reason)`, `notViolated(rule)`, `records(...)` — since a rejection carries an
  evaluation too.
- **A quota is tested by seeding the history it counts**: earlier overrides appended as the actor and
  tagged `RuleTags.overridden(rule)`, exactly as the kernel stores them.

The repository's
[`WithdrawCommandTest`](https://github.com/sliceworkz/eventmodeling/blob/develop/sliceworkz-eventmodeling-examples/src/test/java/org/sliceworkz/eventmodeling/examples/banking/features/withdraw/WithdrawCommandTest.java)
covers every level of the full example.

## Shapes to Avoid

**Throwing a `BusinessException` for a behavioral rule.** It stops at the first violation, cannot be
overridden, and cannot be relaxed without changing code. If people can do otherwise, it is a rule with an
enforcement level.

**Checking a rule for a request that makes no sense.** "Account does not exist" as a strictly enforced
rule puts a decision in front of the user where there is none. It is a `BusinessException`.

**Deciding who may override outside the command.** A role check in the HTTP layer cannot count anything,
and it sits outside the consistency boundary. Pass what the decision needs to the command, and let
`overridableWhen` decide.

**Branching on a check.** `if ( violated ) return;` skips the kernel's judgement: a violation that should
have been reported is silently swallowed, and an override that should have been recorded is not. Check and
raise; the kernel stops what may not go ahead.

**Recording violations by hand instead of `ruleViolations()`.** A hand-built list can disagree with the
kernel's judgement, and the audit then says something that did not happen.

**Treating an evaluation as a reservation.** It is a preview. The execution judges again.

## Design Choices

- **One `execute`, not a validator and an executor.** The check only holds inside the consistency
  boundary the execution's decision models pin, so a separate validator would have to be checked again by
  the executor — two copies of every rule, drifting apart. A kernel that can stop short of the append gives
  the preview for free.
- **No override policy on the builder.** It could not read history, so it could not express a quota or a
  role granted through events; it would sit outside the boundary, so two overrides could race past it; and
  it would escape the command's tests.
- **Overrides on the command, not on `Tracing`.** A tracing continues down a flow — into translators and
  automations — and an override must never reach a later step the user did not see.
- **`QUERY`, not `OPTIONS` or a dry-run flag.** `OPTIONS` describes what a resource supports and is what
  browsers send as a CORS preflight; a flag makes one method on one URI mean two things.

## Quick Reference

| | Where | Outcome |
|---|---|---|
| Request that makes no sense | `BusinessException.when(...)` after `decisionModels(...)` | `BusinessException`, `CommandRejected`; evaluation `REJECTED` |
| Behavioral rule | `context.check(rule, violated, message)` | judged after `execute` by its enforcement level |
| Who may override | `.overridableWhen(authorized, whyNot)`, on decision models | `BLOCKS` with a reason, or `OVERRIDE_REQUIRED` / `OVERRIDDEN` |
| What the user asked for | an `Overrides` component, `implements Overriding` | override requests the kernel judges |
| What to record | `context.ruleViolations()` in the payload; rule tags added by the kernel | audit, quota counts, follow-ups |
| Preview | `evaluate(command, tracing)`; HTTP `QUERY` | `Evaluation`, nothing appended |
| Rejected execution | `RuleViolationException` (a `BusinessException`) | carries the same `Evaluation`; HTTP `422` |
| Follow-up | a todo list or read model on `RuleTags` | justification, deferred enforcement |
| Settling a follow-up | `context.justifies(...)` / `context.enforces(...)` | a link tag, and a `RuleFollowUp` for the payload |
| Rulebook | `builder.businessRules(...)` | announced on `BoundedContextStarting` |
