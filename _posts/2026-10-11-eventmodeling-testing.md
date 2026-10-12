---
layout: post
toc: true
title: Testing Feature Slices
description: The published test bases of sliceworkz-eventmodeling-testing — given/when/then for commands, aggregates, read models, automations, policies, translators, publishers, dispatchers and SQL read models — and how to run every one of them against every event storage you deploy on.
date: 2026-10-11 12:00:00
categories: [Eventmodeling Documentation,Eventmodeling Guides]
tags: [eventmodeling, testing, junit, given when then, archunit, postgresql]
---

Every kind of component a feature slice registers has a published test base: a JUnit 5 class you extend, which builds a real bounded context over a real event storage around each test method and gives you a fluent `given / when / then` for that kind of component. The tests are **synchronous and deterministic** — no polling, no sleeps — and the code they run is the production code: the same command execution, the same automation batch loop, the same publication and policy reaction a deployed processor runs.

And because the bases sit on the eventstore's backend harness, the same test can run against the in-memory store on every build and against PostgreSQL in CI, by changing one annotation.

This guide tests the banking domain of the [quickstart](/posts/eventmodeling-quickstart/), extended as in the other guides: deposits and withdrawals, a welcome bonus automation, the transfers of [Automations and Policies](/posts/eventmodeling-automations-and-policies/), and a policy blocking the cards of a closed account.

## Adding the Dependency

The module's version is managed by the EventModeling BOM. Add it in `test` scope:

```xml
<dependency>
    <groupId>org.sliceworkz</groupId>
    <artifactId>sliceworkz-eventmodeling-testing</artifactId>
    <scope>test</scope>
</dependency>
```

That brings JUnit Jupiter (engine included), the in-memory event storage, the eventstore's backend harness (`@ForEachBackend`), ArchUnit, and H2 plus the PostgreSQL driver and Testcontainers for the SQL read model base. Nothing else is needed for tests that run in memory.

| base | tests | page section |
|---|---|---|
| `CommandTest` | a command, and the business rules it checks | [Commands](#commands-commandtest) |
| `AggregateTest` | an aggregate | [Aggregates](#aggregates-aggregatetest) |
| `LiveModelTest` | a live read model | [Live Read Models](#live-read-models-livemodeltest) |
| `AutomationTest` | an automation with its to-do list | [Automations](#automations-automationtest) |
| `PolicyTest` | a policy and the command it issues | [Policies](#policies-policytest) |
| `TranslatorTest` | translators | [Translators](#translators-translatortest) |
| `PublisherTest` | a publisher | [Publishers](#publishers-publishertest) |
| `DispatcherTest` | a dispatcher | [Dispatchers](#dispatchers-dispatchertest) |
| `sql.SqlReadModelTest` | an SQL read model's projector and query | [SQL Read Models](#sql-read-models-sqlreadmodeltest) |
| `AbstractBoundedContextArchUnitTest` | that slices do not depend on each other | [Architecture Rules](#architecture-rules) |
| `RecordingBoundedContextObserver` | what a context reports to an observer | [Observations](#asserting-on-observations) |

All of them live in `org.sliceworkz.eventmodeling.testing`.

## The Shape Every Base Shares

Every bounded-context base is generic in the context's three event roots and asks you for them:

```java
class DepositCommandTest extends CommandTest<BankingDomainEvent, BankingInboundEvent, BankingOutboundEvent> {

    @Override public Class<BankingDomainEvent> domainEventType ( )    { return BankingDomainEvent.class; }
    @Override public Class<BankingInboundEvent> inboundEventType ( )  { return BankingInboundEvent.class; }
    @Override public Class<BankingOutboundEvent> outboundEventType ( ) { return BankingOutboundEvent.class; }

    // tests
}
```

Before each test method the base builds a fresh, empty storage and a bounded context over it; after the method it terminates the context and releases the storage. Tests never share state. Shredding is always on — an in-memory key store is wired in — so a domain event carrying [personal data](/posts/eventmodeling-personal-data/) works with no setup.

Beyond its fluent API, every test can reach:

| method | gives |
|---|---|
| `kernel()` | the bounded context under test |
| `eventStore()`, `eventStorage()` | the store and storage it was built over, to assert on what was actually written |
| `domainStream()`, `inboundStream()`, `outboundStream()` | ready-made handles on the context's three streams |
| `shreddingKeyStore()` | the key store, to assert on an erasure |
| `waitBecauseOfEventualConsistency(...)` | for an assertion on something projected in the background |
| `backend()` | under `@ForEachBackend`, the backend this invocation runs on |

## Commands: `CommandTest`

### Given, When, Then

```java
private static final AccountId ACC_1 = ACCOUNT.id("acc-1");

@Test
void depositsOnAnOpenAccount ( ) {
    given()
        .event(opened(), ACCOUNT.tags(ACC_1))
        .when(new DepositCommand(ACC_1, 100))
        .then()
        .event(new MoneyDeposited(ACC_1, 100), ACCOUNT.tags(ACC_1));
}

@Test
void rejectsADepositOnAnUnknownAccount ( ) {
    given()
        .when(new DepositCommand(ACC_1, 100))
        .then()
        .businessError("Account acc-1 does not exist");
}
```

- **`given()`** starts a test. `.event(e, tags)` seeds one event of history with its tags.
- **`when(command)`** executes the command through the real kernel — decision models, consistency boundary and all — and records what it appended, or what it threw.
- **`then()`** hands out the assertions.

**Seed history the way production tags it.** A decision model finds its events by tag, so an event seeded without the tags the command would have written is invisible to it. `given(e1, e2)` — the varargs form — seeds events *without* tags, and this test passes for exactly that reason:

```java
@Test
void anUntaggedEventIsInvisibleToTheDecisionModel ( ) {
    given(opened())                       // no tags: the decision model never sees it
        .when(new DepositCommand(ACC_1, 100))
        .then()
        .businessError("Account acc-1 does not exist");
}
```

Use the untagged form only for events no decision model queries by tag. Everywhere else, `.event(e, tags)`.

### What You Can Assert

| assertion | passes when |
|---|---|
| `event(e)` | exactly one event was appended, equal to `e` |
| `event(e, tags)` | exactly one event, equal to `e`, carrying *at least* `tags` |
| `events(e1, e2, ...)` | exactly these events, in this order |
| `noEvents()` | the command succeeded and appended nothing |
| `businessError(message)` | it threw a `BusinessException` with exactly this message |
| `businessError()` | it threw a `BusinessException` |
| `error(Type.class)` / `error(Type.class, message)` | it threw a `Type` (with this message) |
| `error(message)` | it threw something whose **root cause** has this message |
| `rulesViolated()` | the kernel rejected it on its business rules — see below |

Events are compared field by field, through their JSON rendering. The tag check is containment: the framework adds its own `x-*` tracing tags to every event, so assert on the tags you care about and ignore the rest.

**Assert a rule by type.** `businessError(...)` is the assertion to reach for: it holds the command to the one distinction [Where a Validation Goes](/posts/eventmodeling-where-validations-go/) asks it to keep — a `BusinessException` for a rule, never an `IllegalStateException`, which is what a bug throws. A type assertion judges the exception that came out of the execution, exactly as a `catch` block would see it, and deliberately does not search the cause chain. `error(message)` is the one that looks at the root cause, which makes it useful across the wrapping the framework does on some paths, and blind to the type.

### Minted Ids and Other Unpredictable Values

`OpenAccountCommand` mints the account id with `ACCOUNT.newId()`, so the test cannot know it. `IGNORE_ID()` marks the field: it must be present and parse as a UUID, and only its value is not compared:

```java
@ForEachBackend
void openingAnAccountRaisesAccountOpened ( ) {
    given()
        .when(new OpenAccountCommand(ALICE))
        .then()
        .event(new AccountOpened(ACCOUNT.id(IGNORE_ID()), ALICE, LocalDate.now()), CUSTOMER.tags(ALICE));
}
```

The marker goes through the entity's own factory, `ACCOUNT.id(IGNORE_ID())`, so it lands on the record's `value` line. The tag assertion names only the customer's tag, since the account's tag value is the minted id. `IGNORE_TEXT()` skips a field entirely — presence and value — for a value that is not a UUID.

### A Command With a Result

A `CommandWithResult` runs through the same `when(...)`; its response is on the result, typed as `TestResultWithResponse`:

```java
@Test
void theResponseIsTheNewAccountsId ( ) {
    var result = given()
        .when(new OpenAccountWithResultCommand(CUSTOMER.id("alice")))
        .then();
    AccountId accountId = ((TestResultWithResponse<BankingDomainEvent, AccountId>) result).response();
    result.event(new AccountOpened(accountId, CUSTOMER.id("alice"), LocalDate.now()), ACCOUNT.tags(accountId));
}
```

The response is what lets the event assertion name the minted id exactly.

### Business Rules

A command checking [business rules with enforcement levels](/posts/eventmodeling-business-rules/) gets three more verbs:

- **`as(actor)`** sets the actor from there on: events seeded after it carry the actor's tag, and the command runs for that actor — what `context.actor()` answers.
- **`whenEvaluated(command)`** evaluates instead of executing — the preview a front end asks for — and fails the test if anything was appended. **`thenEvaluation()`** hands out the assertions.
- **`then().rulesViolated()`** asserts that an *execution* was rejected on its rules, and hands out the same assertions, because a rejected execution carries the evaluation a preview would have answered.

With a strictly enforced `MAXIMUM_WITHDRAWAL` and a `WITHDRAWAL_DESCRIBED` guideline:

```java
@Test
void aWithdrawalAboveTheMaximumIsBlocked ( ) {
    given()
        .event(opened(), ACCOUNT.tags(ACC_1))
        .event(new MoneyDeposited(ACC_1, 10_000), ACCOUNT.tags(ACC_1))
        .when(new WithdrawCommand(ACC_1, 6_000, "car"))
        .then()
        .rulesViolated()
        .blockedBy(MAXIMUM_WITHDRAWAL);
}

@Test
void aPreviewShowsTheGuidelineWithoutAppendingAnything ( ) {
    given()
        .event(opened(), ACCOUNT.tags(ACC_1))
        .event(new MoneyDeposited(ACC_1, 1_000), ACCOUNT.tags(ACC_1))
        .as("alice")
        .whenEvaluated(new WithdrawCommand(ACC_1, 100, ""))
        .thenEvaluation()
        .wouldSucceed()
        .judged(WITHDRAWAL_DESCRIBED, Verdict.ADVISED)
        .notViolated(MAXIMUM_WITHDRAWAL);
}

@Test
void anAccountThatDoesNotExistIsRejectedInThePreviewToo ( ) {
    given()
        .whenEvaluated(new WithdrawCommand(ACC_1, 100, "rent"))
        .thenEvaluation()
        .rejected("Account acc-1 does not exist");
}
```

| evaluation assertion | checks |
|---|---|
| `wouldSucceed()` | the execution would go ahead |
| `needsOverrideOf(rules...)` | it would go ahead only if exactly these rules were overridden |
| `blockedBy(rules...)` | it is blocked, by exactly these rules |
| `rejected(message)` | the command threw a `BusinessException` with this message before the rules were judged |
| `judged(rule, verdict)` | this rule was judged with this verdict (`BLOCKS`, `OVERRIDE_REQUIRED`, `OVERRIDDEN`, `DEFERRED`, `ADVISED`) |
| `notOverridable(rule, reason)` | this rule blocks, for this reason |
| `notViolated(rule)` | this rule was not violated |
| `records(violations...)` | exactly these violations would be recorded |

An execution that **goes ahead** with violations is asserted through its event: the `RuleViolation`s the command put in its payload, and the rule tag the kernel added:

```java
@Test
void aWithdrawalWithoutDescriptionGoesAheadAndRecordsTheGuideline ( ) {
    given()
        .event(opened(), ACCOUNT.tags(ACC_1))
        .event(new MoneyDeposited(ACC_1, 1_000), ACCOUNT.tags(ACC_1))
        .when(new WithdrawCommand(ACC_1, 100, ""))
        .then()
        .event(new MoneyWithdrawn(ACC_1, 100, "", List.of(new RuleViolation(WITHDRAWAL_DESCRIBED.id(),
                    EnforcementLevel.GUIDELINE, Disposition.GUIDELINE_NOT_FOLLOWED, "No description given", null))),
               Tags.of(RuleTags.notFollowed(WITHDRAWAL_DESCRIBED)));
}
```

A quota the command counts — "three overdrafts per teller per day" — is tested by seeding the history it counts, `as(...)` the actor and tagged as the kernel tags an override. [`WithdrawCommandTest`](https://github.com/sliceworkz/eventmodeling/blob/develop/sliceworkz-eventmodeling-examples/src/test/java/org/sliceworkz/eventmodeling/examples/banking/features/withdraw/WithdrawCommandTest.java) in the examples covers every enforcement level that way.

## Aggregates: `AggregateTest`

An [aggregate](/posts/aggregates-and-dcb/) is tested within its identity. The fourth type parameter is the aggregate class:

```java
class CardAggregateTest extends AggregateTest<BankingDomainEvent, BankingInboundEvent, BankingOutboundEvent, CardAggregate> {

    // the three event type methods, plus:
    @Override public Class<CardAggregate> getAggregateClass ( ) { return CardAggregate.class; }

    @Test
    void anIssuedCardIsBlocked ( ) {
        given(CARD.tags(CARD_A))
            .events(new CardIssued(ACC_1, CARD_A))
            .when(card -> card.block("reported stolen"))
            .then()
            .event(new CardBlocked(ACC_1, CARD_A, "reported stolen"));
    }

    @Test
    void aBlockedCardIsNotBlockedTwice ( ) {
        given(CARD.tags(CARD_A))
            .events(new CardIssued(ACC_1, CARD_A), new CardBlocked(ACC_1, CARD_A, "reported lost"))
            .when(card -> card.block("reported stolen"))
            .then()
            .businessError("Card card-a is already blocked");
    }
}
```

`given(identity)` names the aggregate; `.events(...)` seeds history tagged with that identity (`.event(e, tags)` seeds with other tags); `when(...)` loads the aggregate and hands it to your action. The assertions are the command ones, without the rules.

## Live Read Models: `LiveModelTest`

Name the read model class; the base registers it `.live()`. `when(args...)` reads it with those constructor arguments, the way `read(AccountDetailsReadModel.class, accountId)` would:

```java
class AccountDetailsReadModelTest extends LiveModelTest<BankingDomainEvent, BankingInboundEvent, BankingOutboundEvent> {

    @Override
    public Class<? extends ReadModel<BankingDomainEvent>> getLiveModelClass ( ) {
        return AccountDetailsReadModel.class;
    }

    @Test
    void showsTheBalanceOfOneAccount ( ) {
        given()
            .event(new AccountOpened(ACC_1, CUSTOMER.id("alice"), LocalDate.of(2026, 1, 5)), ACCOUNT.tags(ACC_1))
            .event(new MoneyDeposited(ACC_1, 100), ACCOUNT.tags(ACC_1))
            .event(new AccountOpened(ACC_2, CUSTOMER.id("bob"), LocalDate.of(2026, 1, 6)), ACCOUNT.tags(ACC_2))
            .event(new MoneyDeposited(ACC_2, 999), ACCOUNT.tags(ACC_2))
            .when(ACC_1)
            .then()
            .liveModel(model -> ((AccountDetailsReadModel) model).details().orElseThrow())
            .is(new AccountDetails("acc-1", "alice", 100, false));
    }

    @Test
    void anUnknownAccountHasNoDetails ( ) {
        given()
            .when(ACC_1)
            .then()
            .liveModel(model -> ((AccountDetailsReadModel) model).details().isPresent())
            .satisfies(present -> assertEquals(false, present));
    }
}
```

`liveModel(mapper)` picks the part of the read model to assert on — usually the value it exposes — and **must come first**: `is(expected)` compares that mapped value field by field, and `satisfies(assertion)` hands it to your own assertion. Seeding a second account is worth the two lines: it proves the read model's query selects by its tag and not by luck.

## Automations: `AutomationTest`

`AutomationTest` tests an automation together with its to-do list. The batch loop it runs is the production one, over a real `AutomationContext`, so idempotency keys and `onFailure` behave as deployed. The test constructs the automation itself, which is where a port gets its fake:

```java
class GrantWelcomeBonusAutomationTest
        extends AutomationTest<AccountAwaitingBonus, BankingDomainEvent, BankingInboundEvent, BankingOutboundEvent> {

    @Override
    public Automation<AccountAwaitingBonus, BankingDomainEvent, BankingOutboundEvent> automation ( ) {
        return new GrantWelcomeBonusAutomation(new AccountsAwaitingWelcomeBonusTodoList());
    }

    @Test
    void aNewAccountGetsItsBonusOnce ( ) {
        given()
            .event(opened(), ACCOUNT.tags(ACC_1))
            .expectTodoItems(new AccountAwaitingBonus(ACC_1))
            .whenBatchRuns()
                .itemsHandled(1)
                .event(new WelcomeBonusGranted(ACC_1, GrantWelcomeBonusCommand.BONUS), ACCOUNT.tags(ACC_1))
            .and()
            .expectNoTodoItems();
    }

    @Test
    void aRedeliveredItemGrantsNothingTwice ( ) {
        given()
            .event(opened(), ACCOUNT.tags(ACC_1))
            .whenBatchRuns()
                .itemsHandled(1)
            .and()
            .whenItemsAreRedelivered()
                .itemsHandled(1)
                .noEvents();
    }
}
```

**How a round works, and why it matches production.** `whenBatchRuns()` first projects everything appended so far into the to-do list — what the list's own projector does on its thread in a deployment — then runs exactly **one** batch. It deliberately does not project afterwards: a batch's events reach the to-do list at the start of the *next* round, exactly as deployed. That is why `expectNoTodoItems()` after `.and()` sees the item gone — it projects first.

**`whenItemsAreRedelivered()`** is the same round *without* the catch-up, so the to-do list re-offers the items the previous batch already handled: the crash between the append and the bookmark. `.noEvents()` there is the one-line proof that the command's key, derived from the account (`"welcome-bonus/" + accountId`), makes the repeat a no-op. A key derived from the attempt would fail this test.

| batch assertion | checks |
|---|---|
| `itemsHandled(n)` / `itemsFailed(n)` | how many items `handle` completed / threw on |
| `events(e...)`, `event(e, tags)`, `noEvents()` | what this batch appended to the domain stream |
| `automationStopped()` / `automationStillRunning()` | whether `onFailure` answered `STOP_AUTOMATION` |
| `and()` | back to the definition, for the next round or `expectTodoItems(...)` |

Failures are scripted through the fake. A gateway that answers from a queue covers each `onFailure` branch:

```java
private final Deque<Supplier<String>> answers = new ArrayDeque<>();
private final TransferGateway gateway = ( toIban, amountInCents, key ) -> answers.pop().get();

@Test
void anUnexpectedFailureStopsTheAutomation ( ) {
    answers.add(() -> { throw new IllegalStateException("adapter bug"); });
    given()
        .event(new TransferRequested(T_1, ACC_1, "BE01", 100), TRANSFER.tags(T_1))
        .whenBatchRuns()
            .automationStopped();
}
```

After a batch that stopped the automation, the next `whenBatchRuns()` refuses to run until `restartAutomation()` — as in production, where a stopped automation waits for an operator.

> **Never register the automation or its to-do list on the builder.** The base owns both instances and drives them itself; a registered automation would also run on a real processor thread, racing the synchronous rounds, and every assertion turns non-deterministic.
{: .prompt-warning }

**Seed retry delays, never wait them out.** "Declined twice, then abandoned" is tested by seeding the failure events a past run would have recorded, with a due time already reached, and running one batch on top. [`ExecutePaymentAutomationTest`](https://github.com/sliceworkz/eventmodeling/blob/develop/sliceworkz-eventmodeling-examples/src/test/java/org/sliceworkz/eventmodeling/examples/payments/features/executepayment/ExecutePaymentAutomationTest.java) covers every branch of the payments automation that way, and [`PaymentsToExecuteTodoListTest`](https://github.com/sliceworkz/eventmodeling/blob/develop/sliceworkz-eventmodeling-examples/src/test/java/org/sliceworkz/eventmodeling/examples/payments/features/executepayment/PaymentsToExecuteTodoListTest.java) the to-do list on its own.

## Policies: `PolicyTest`

`PolicyTest` hands seeded events to a [policy](/posts/eventmodeling-automations-and-policies/) through the same reaction a deployed policy's processor runs, and executes the command it issues through the test's bounded context — so the command decides on its decision models over the seeded history:

```java
class BlockCardsOfClosedAccountPolicyTest extends PolicyTest<BankingDomainEvent, BankingInboundEvent, BankingOutboundEvent> {

    @Override
    public Policy<BankingDomainEvent> policy ( ) {
        return new BlockCardsOfClosedAccountPolicy();
    }

    @Test
    void aRedeliveredEventIsNotReactedToAgain ( ) {
        given(new AccountOpened(ACC_1, CUSTOMER.id("alice"), LocalDate.of(2026, 1, 5)), ACCOUNT.tags(ACC_1))
            .and(new CardIssued(ACC_1, CARD_A), Tags.of(ACCOUNT.tag(ACC_1), CARD.tag(CARD_A)))
            .and(new AccountClosed(ACC_1), ACCOUNT.tags(ACC_1))
        .whenReacting()
            .events(new CardBlocked(ACC_1, CARD_A, "account closed"))
        .and()
        .whenRedelivered()
            .noEvents();
    }
}
```

- **`whenReacting()`** hands over every matching event the policy has not seen yet. A second `whenReacting()` after more `.and(...)` events hands over only the new ones.
- **`whenRedelivered()`** hands over everything again — the crash-or-failover case — and a correct policy raises nothing.
- The assertions: `events(...)`, `event(e, tags)`, `noEvents()`, `ignored()` (every event handed over was answered with no command), `rejected(message)`, `failed(Type.class)`.

A rejection follows the policy's rejection handling. By default it **stalls**, as a policy registered `.stallOnRejection()` would; override `onRejection()` to test one registered `.skipRejections()`. Take a policy whose command is refused — one that deposits onto an account the moment it is closed, which `DepositCommand` rejects:

```java
@Test
void aRejectionStallsThePolicy ( ) {
    var result = given(new AccountOpened(ACC_1, CUSTOMER.id("alice"), LocalDate.of(2026, 1, 5)), ACCOUNT.tags(ACC_1))
        .and(new AccountClosed(ACC_1), ACCOUNT.tags(ACC_1))
        .whenReacting()
        .rejected("Account acc-1 is closed");
    assertNotNull(result.stalledOn());            // the event the policy is stuck on
}
```

```java
// in a test class that skips rejections
@Override protected PolicyRejectionHandling onRejection ( ) { return PolicyRejectionHandling.SKIP; }

@Test
void aRejectionIsSkipped ( ) {
    given(new AccountOpened(ACC_1, CUSTOMER.id("alice"), LocalDate.of(2026, 1, 5)), ACCOUNT.tags(ACC_1))
        .and(new AccountClosed(ACC_1), ACCOUNT.tags(ACC_1))
        .whenReacting()
        .rejected("Account acc-1 is closed")
        .noEvents();
}
```

Never register the policy on the builder either. [`BlockCardsOfAccountPolicyTest`](https://github.com/sliceworkz/eventmodeling/blob/develop/sliceworkz-eventmodeling-examples/src/test/java/org/sliceworkz/eventmodeling/examples/banking/features/blockcardsofaccount/BlockCardsOfAccountPolicyTest.java) is the worked example.

## Translators: `TranslatorTest`

`TranslatorTest` registers the translators you name and runs them through the interactive `translate(...)` path, synchronously:

```java
class PartnerDepositTranslatorTest extends TranslatorTest<BankingDomainEvent, BankingInboundEvent, BankingOutboundEvent> {

    @Override
    public List<Translator<BankingInboundEvent, BankingDomainEvent>> translators ( ) {
        return List.of(new PartnerDepositTranslator());
    }

    @Test
    void aPartnerDepositBecomesADeposit ( ) {
        given()
            .when(new PartnerDepositReceived("acc-1", 250))
            .then()
            .event(new MoneyDeposited(ACCOUNT.id("acc-1"), 250), ACCOUNT.tags(ACCOUNT.id("acc-1")));
    }
}
```

Two properties of that path are asserted on every test for free: the inbound event is **not** persisted, and an inbound event no translator claims makes `then().noTranslatorRegistered()` pass rather than vanishing. The interactive path matches translators on the inbound event's *type*; the translation logic is the same one the asynchronous `incoming(...)` path runs. See [Translators, Publishers and Dispatchers](/posts/eventmodeling-translators-publishers-dispatchers/).

## Publishers: `PublisherTest`

`PublisherTest` drives a publisher over seeded domain events through the same publication code a deployed publisher runs — the idempotency keys, the `x-published-from` tag and the first-publication-stands rule are the production ones:

```java
@Test
void anOpenedAccountIsAnnouncedOnceAndOnlyOnce ( ) {
    given(new AccountOpened(ACC_1, CUSTOMER.id("alice"), LocalDate.of(2026, 1, 5)), ACCOUNT.tags(ACC_1))
    .whenPublished()
        .published(new AccountAnnounced(ACC_1), ACCOUNT.tags(ACC_1))
    .and()
        .and(new AccountOpened(ACC_2, CUSTOMER.id("bob"), LocalDate.of(2026, 1, 6)), ACCOUNT.tags(ACC_2))
    .whenPublished()
        .published(new AccountAnnounced(ACC_2))
    .and()
    .whenRepublished()
        .nothingPublished();
}
```

`whenPublished()` hands over what the publisher has not seen yet, so the second round publishes only `ACC_2`. `whenRepublished()` hands every event over again — a redelivery — and the store swallows what was published before. The live read models a publisher reads are projected as it asks for them, bounded as a deployment bounds them, with nothing to register.

## Dispatchers: `DispatcherTest`

A dispatcher is a projection over the outbound stream, and `DispatcherTest` drives it as one. `given(...)` seeds outbound events as fixture data; the test asserts on the fake the dispatcher sends to:

```java
private final List<String> sent = new ArrayList<>();

@Override
public Dispatcher<BankingOutboundEvent> dispatcher ( ) {
    return new AccountAnnouncedDispatcher(( topic, message ) -> sent.add(topic + ":" + message));
}

@Test
void anAnnouncementReachesTheBusOncePerRound ( ) {
    given(new AccountAnnounced(ACCOUNT.id("acc-1")))
        .whenDispatched().delivered(1)
        .and()
        .whenDispatched().nothingDelivered()
        .and()
        .whenRedeliveredFromTheStart().delivered(1);
    assertEquals(List.of("accounts:acc-1", "accounts:acc-1"), sent);
}
```

`whenDispatched()` keeps its cursor across rounds, as the bookmark does, so a second round delivers only what is new. `whenRedeliveredFromTheStart()` starts from zero — a lost bookmark — and the test faces up to what the dispatcher does with a duplicate: here, the bus received the announcement twice. A dispatcher is where duplicate publishing costs most, so this is the test to write before deciding whether the receiving side needs a key.

## SQL Read Models: `SqlReadModelTest`

[Durable SQL read models](/posts/eventmodeling-sql-read-models/) write into a database of their own, and `sql.SqlReadModelTest` tests the projector and the query side against it — not against the event storage. The test methods sit in an abstract inner class, and a `@Nested` class per database runs them:

```java
class AccountListTest extends SqlReadModelTest<BankingDomainEvent> {

    abstract static class Tests extends AbstractSqlReadModelTests<BankingDomainEvent> {

        @Override protected SqlReadModelProjector<BankingDomainEvent> createProjector ( DataSource ds ) { return new AccountListProjector(ds); }
        @Override protected SqlReadModelQuery createQuery ( DataSource ds ) { return new AccountListQuery(ds); }
        @Override protected String[] tablesToDrop ( ) { return new String[] { "bank_accounts", "bank_projection_bookmark" }; }

        @Test
        void listsTheAccountsOfACustomer ( ) {
            projectEvents(
                new AccountOpened(ACCOUNT.id("acc-1"), CUSTOMER.id("alice"), LocalDate.of(2026, 1, 5)),
                new AccountOpened(ACCOUNT.id("acc-2"), CUSTOMER.id("bob"), LocalDate.of(2026, 1, 6)));
            AccountListQuery query = query();
            assertEquals(List.of("acc-1"), query.accountsOf("alice"));
            assertEquals(true, projectedUpTo().isPresent());
        }
    }

    @Nested class OnH2 extends Tests {
        @Override protected DataSource createDataSource ( ) { return h2DataSource(); }
    }

    @Nested class OnPostgres extends Tests {
        @Override protected DataSource createDataSource ( ) { return postgresDataSource(); }
    }
}
```

`projectEvents(...)` runs the events through the projector's real batch lifecycle, closing the batch the way the framework does, so the projector records its own position too. `projectedUpTo()` is that position — where a restarted projector would resume — and `restartedProjector()` builds a second projector over the same database, to prove re-projecting changes nothing. `h2DataSource()` is a fresh in-memory database per call; `postgresDataSource()` (or `postgresDataSource(image)`) a shared Testcontainers PostgreSQL, so that class needs a Docker daemon. List the projector's own bookmark table in `tablesToDrop()` too, or the second test resumes where the first stopped.

## Running Every Test Against Every Storage

### `@Test` or `@ForEachBackend`

Which storage a test runs on depends on how its method is annotated:

| annotation | runs |
|---|---|
| `@Test` | once, against an in-memory store. Nothing to configure |
| `@ForEachBackend` | once per registered `EventStoreBackend`, each reported under its backend's name: `openingAnAccountRaisesAccountOpened [postgres:18]` |

`@ForEachBackend` is `org.sliceworkz.eventstore.testing.ForEachBackend`, the eventstore's own — see [Testing Your Application](/posts/eventstore-testing/) for the harness behind it. It works on every bounded-context base: `CommandTest`, `AggregateTest`, `LiveModelTest`, `AutomationTest`, `PolicyTest`, `TranslatorTest`, `PublisherTest` and `DispatcherTest`.

**Use `@ForEachBackend` where the storage could change the answer**: anything touching consistency boundaries, tags, idempotency keys or ordering — a DCB conflict under a real advisory lock, a tag round trip through PostgreSQL's `text[]`, a redelivery swallowed by a stream-scoped unique index. **Use `@Test` where the scenario is about your own code** and no storage can change it; it costs no container.

### Registering the Backends

The backend set is data, not code. List the ones you want in

```
src/test/resources/META-INF/services/org.sliceworkz.eventstore.testing.EventStoreBackend
```

```
org.sliceworkz.eventstore.testing.backend.InMemoryBackend
org.sliceworkz.eventstore.testing.backend.InMemoryFsBackend
org.sliceworkz.eventstore.testing.backend.Postgres17Backend
org.sliceworkz.eventstore.testing.backend.Postgres18Backend
```

`InMemoryBackend` needs nothing further. The others need their storage on the test classpath, which `sliceworkz-eventstore-testing` declares optional so that a project using only the in-memory store does not inherit a database driver:

```xml
<dependency>
    <groupId>org.sliceworkz</groupId>
    <artifactId>sliceworkz-eventstore-infra-inmem-fs</artifactId>   <!-- for InMemoryFsBackend -->
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.sliceworkz</groupId>
    <artifactId>sliceworkz-eventstore-infra-postgres</artifactId>   <!-- for the PostgreSQL backends -->
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>com.github.f4b6a3</groupId>
    <artifactId>uuid-creator</artifactId>                           <!-- PostgreSQL 17 and older -->
    <version>6.1.1</version>
    <scope>test</scope>
</dependency>
```

The PostgreSQL backends run in Testcontainers, so they need a Docker daemon. One container is started per image per JVM and shared; isolation between tests is a schema drop and recreate, not a fresh container. Under `@ForEachBackend` the key store is the backend's own — the SQL table on PostgreSQL — so a shredding test exercises the key store you deploy.

### Narrowing a Run

```bash
mvn test -Deventstore.testing.backends=inmem                # skip the containers entirely
mvn test -Deventstore.testing.backends=inmem,postgres:18
```

Handy locally and in a pre-commit hook; leave the full set to CI. The names are the backends' own: `inmem`, `inmem-fs`, `postgres:16`, `postgres:17`, `postgres:18`.

> The bounded-context bases build the storage themselves through the harness, around every test method. A test never builds a storage of its own: reach the one it runs on through `eventStorage()`.
{: .prompt-tip }

## Architecture Rules

`AbstractBoundedContextArchUnitTest` checks two rules with ArchUnit, over every class in the package of the test that extends it and below — test classes included. Extend it with an empty class in your application's root package:

```java
package com.example.banking;

class BankingArchitectureTest extends AbstractBoundedContextArchUnitTest { }
```

- **Feature packages do not depend on each other.** A feature package is any package holding a `Slice` implementation, subpackages included. A slice that imports another slice's class fails:
  ```
  Class com.example.banking.features.checkcustomer.CheckCustomerFeatureSlice depends on
      com.example.banking.features.openaccount.CustomerDirectory from different feature package
  ```
  The fix is to move what both need — a [port](/posts/eventmodeling-ports-and-adapters/), a shared decision model — into a package that holds no slice, such as `com.example.banking.ports`.
- **A value object with a static factory is created through it.** A record declaring a static method that returns its own type must not be constructed with `new` from outside — the rule that keeps validation in the factory and the canonical constructor lenient (see [Where a Validation Goes](/posts/eventmodeling-where-validations-go/)).

Because test classes are scanned too, keep a test that exercises two slices together — a scenario across features — in a package of its own rather than inside one of the feature packages.

## Asserting on Observations

`RecordingBoundedContextObserver` is a `BoundedContextObserver` that keeps everything it is told, for a test asserting on what a metrics or tracing binding would see. Build a context with it as the observer:

```java
RecordingBoundedContextObserver observer = new RecordingBoundedContextObserver();
Banking banking = BoundedContext.newBuilder(Banking.class)
    .name("banking").eventStorage(storage).instance(InstanceFactory.determine("t"))
    .observer(observer)
    .build();
banking.start();

banking.execute(new OpenAccountCommand(CUSTOMER.id("alice")));

Recording execution = observer.last(Observation.CommandExecution.class);
Outcome.Executed executed = execution.outcome(Outcome.Executed.class);
assertEquals(1, executed.appended().size());
assertEquals(List.of(), observer.violations());   // every scope answered once and closed in order
```

Each `Recording` keeps the recording it was started inside of (`parent()`), so a test can check nesting — that a port call made by an automation arrives under its `Observation.AutomationRun`, say. `violations()` lists every breach of the observer contract the framework made; asserting it empty checks the framework along the way. `clear()` forgets what was recorded, to observe one step on its own. See [Observability and Tracing](/posts/eventmodeling-observability/).

## Worked Examples

The examples module carries a test per pattern, written against these bases:

| test | shows |
|---|---|
| [`WithdrawCommandTest`](https://github.com/sliceworkz/eventmodeling/blob/develop/sliceworkz-eventmodeling-examples/src/test/java/org/sliceworkz/eventmodeling/examples/banking/features/withdraw/WithdrawCommandTest.java) | every enforcement level, `as(...)`, previews and a quota |
| [`ExecutePaymentAutomationTest`](https://github.com/sliceworkz/eventmodeling/blob/develop/sliceworkz-eventmodeling-examples/src/test/java/org/sliceworkz/eventmodeling/examples/payments/features/executepayment/ExecutePaymentAutomationTest.java) | every `onFailure` branch, deferred retries seeded rather than waited out |
| [`ReportExcessBalanceAutomationTest`](https://github.com/sliceworkz/eventmodeling/blob/develop/sliceworkz-eventmodeling-examples/src/test/java/org/sliceworkz/eventmodeling/examples/banking/features/excessbalance/ReportExcessBalanceAutomationTest.java) | a redelivered item swallowed by its key |
| [`BlockCardsOfAccountPolicyTest`](https://github.com/sliceworkz/eventmodeling/blob/develop/sliceworkz-eventmodeling-examples/src/test/java/org/sliceworkz/eventmodeling/examples/banking/features/blockcardsofaccount/BlockCardsOfAccountPolicyTest.java) | a policy, its redelivery, and a later trigger blocking nothing twice |

The module's own [README](https://github.com/sliceworkz/eventmodeling/blob/develop/sliceworkz-eventmodeling-testing/README.md) has the reference for each base.
