---
layout: post
toc: true
title: Personal Data and Erasure
description: Declaring personal data as Shreddable values in domain events, configuring where the keys live, erasing a person or one category of their data, and what erasure does not reach.
date: 2026-10-12 15:00:00
categories: [Eventmodeling Documentation,Eventmodeling Deployment]
tags: [eventmodeling, gdpr, personal data, erasure, crypto-shredding, shreddable, privacy]
---

An event log is append-only, and a request to be forgotten asks you to remove something from it. The eventstore resolves that conflict with **crypto-shredding**: personal data is stored encrypted under a key held per person, and erasing the person destroys the key. The events stay byte-identical; what they held about that person can no longer be read — in the table, in the write-ahead log, on the replicas and in every backup.

The mechanism — the sealed envelope, the key stores, the audit view, rotation, readers that may not see personal data — is the eventstore's and is explained in [Erasing Personal Data](/posts/eventstore-erasing-personal-data/). This guide is about using it from a bounded context: what goes into a domain event, what goes on the builder, who may erase, and the part no framework can do for you — the copies your read models hold.

## Declaring Personal Data in a Domain Event

A record component that holds personal data is declared `Shreddable<T>` and bound to a **data subject** — the person the data is about:

```java
public interface BankingDomain {

    record AccountId ( String value ) implements EntityId { }
    record CustomerId ( String value ) implements EntityId { }

    Entity<AccountId>  ACCOUNT  = Entity.of("account", AccountId::new);
    Entity<CustomerId> CUSTOMER = Entity.of("customer", CustomerId::new);

    record Address ( String street, String city ) { }

    /** The data subject a customer's personal data is sealed for: the customer id, never a name or an email. */
    static DataSubject customerSubject ( CustomerId customerId ) {
        return DataSubject.of("customer", customerId.value());
    }

    sealed interface BankingDomainEvent {
        record AccountOpened ( AccountId accountId, CustomerId customerId, LocalDate date ) implements BankingDomainEvent { }
        record CustomerRegistered ( CustomerId customerId,
                                    Shreddable<String> name,        // category "default"
                                    Shreddable<Address> address )   // category "address"
               implements BankingDomainEvent { }
        record CustomerForgotten ( CustomerId customerId ) implements BankingDomainEvent { }
        // ...
    }
    // ...
}
```

Three decisions are made here, once, and are hard to change later:

- **What is personal.** `customerId` is pseudonymous — it survives erasure and keeps every event queryable and every account reconcilable. `name` and `address` are personal and wrapped. A whole object can be one `Shreddable` (`Shreddable<Address>`), and a `Shreddable` can sit anywhere in the payload: in nested records, list elements, map values.
- **Whose it is.** A `DataSubject` is a type and an id. The id is stored **in the clear** on every sealed value, so it must not itself be personal data: use the customer number, never an email address. A helper such as `customerSubject(...)` keeps that mapping in one place, the way an `Entity` keeps the tag key in one place (see [Entities, Typed Ids and Tags](/posts/eventmodeling-entities-and-tags/)).
- **Which category.** A subject's data is sealed per **category** — `DataSubject.of(type, id)` is the `default` one, and `withCategory("address")` another. Each category has its own key, so "forget the address but keep the name" is erasing one category. A category is fixed when the event is written: a value sealed under `default` cannot later be moved to another category without rewriting the event. Keep it to a handful per subject, chosen by what you will want to erase or withhold separately.

A component that was a plain field when its events were written cannot be turned into a `Shreddable` afterwards — the stored value carries no subject. Read the old shape through a `@LegacyEvent` upcaster, or migrate the events with the eventstore's importer.

> A shredded value is **never null**. A record whose constructor rejects nulls still builds after its data is gone, so erasure does not turn events into poison events. Keep payload records' canonical constructors lenient anyway (see [Where a Validation Goes](/posts/eventmodeling-where-validations-go/)).
{: .prompt-info }

## Raising It

A command wraps the values when it raises the event. Nothing else in the command changes:

```java
public record RegisterCustomerCommand ( CustomerId customerId, String name, Address address )
        implements Command<BankingDomainEvent> {

    @Override
    public void execute ( CommandContext<BankingDomainEvent, BankingDomainEvent> context ) {
        DataSubject customer = BankingDomain.customerSubject(customerId);
        context.noDecisionModels()
               .raiseEvent(new CustomerRegistered(customerId,
                               Shreddable.of(name, customer),
                               Shreddable.of(address, customer.withCategory("address"))),
                           CUSTOMER.tags(customerId));
    }
}
```

On append, the store seals each value under the key held for its subject and category — minting one the first time — and tags the event with the key ids (`dek:...` tags), so "every event holding data under this key" is an ordinary tag query. Commands, decision models, read models, automations, translators, publishers and dispatchers all see a `Shreddable` as an ordinary value in the payload. There is nothing to configure per slice.

## Configuring Where the Keys Live

`shredding(...)` on the builder is the whole setup. The shipped AES-256-GCM codec is applied for you, so the only choice is where the keys are kept:

```java
// production: keys in <prefix>shredding_keys, in the same PostgreSQL database as the events
.shredding(PostgresShreddingKeyStore.on(dataSource, "banking_"))

// local development on file-backed storage: keys in a file beside the events
.shredding(new InMemoryFsShreddingKeyStore(directory))

// tests and throwaway runs: keys die with the JVM
.shredding(new InMemoryShreddingKeyStore())
```

| key store | lives | use it with |
|---|---|---|
| `PostgresShreddingKeyStore.on(dataSource, prefix)` | a table created by the ordinary schema machinery, next to the events | `PostgresEventStorage` |
| `new InMemoryFsShreddingKeyStore(directory)` | `keys.jsonl` in the directory | `InMemoryFsEventStorage` — events that outlive the process need keys that do too |
| `new InMemoryShreddingKeyStore()` | the heap | `InMemoryEventStorage` |

Match the key store's lifetime to the events'. An in-memory key store beside file-backed events makes every value written by a previous run read as **erased** on the next start — the keys are gone. And every process reading a store needs the same key store: one pointed at the wrong prefix or directory finds keys it never held, and throws rather than reporting the data as erased (see [Erased Means Erased](/posts/eventstore-erasing-personal-data/#erased-means-erased-unavailable-means-throw)).

For a codec of your own — one that keeps key material inside an HSM so it never enters the JVM — use `shredding(ShreddingCodec)` instead. It is also how a context reads a **narrower** view than its storage unlocks: a reporting service that may read names but never addresses gets a codec restricted to the `default` category, and every address reads as *withheld*:

```java
ShreddingCodec namesOnly = AesGcmShreddingCodec
        .over(PostgresShreddingKeyStore.on(dataSource, "banking_"))
        .restrictedTo(Set.of("default"));

Banking reporting = BoundedContext.newBuilder(Banking.class)
        .name("banking")
        .eventStorage(storage)
        .shredding(namesOnly)
        // ...
        .build();
```

`ShreddingCodec.withholdingAll()` is the extreme: typed events, none of the personal data. See [Readers That May Not See Personal Data](/posts/eventstore-erasing-personal-data/#readers-that-may-not-see-personal-data).

### The codec travels with the storage

A storage builder takes `.shredding(...)` too, and the codec then travels with the storage: a bounded context built on it without a `shredding(...)` of its own uses the storage's.

```java
// the no-argument shredding() keeps keys in <prefix>shredding_keys, on the builder's DataSource
EventStorage storage = PostgresEventStorage.newBuilder()
        .dataSource(dataSource)
        .prefix("banking_")
        .shredding()
        .build();
```

So configure it in one of two places: on the storage, where every context and tool on that storage shares it, or on the context. A codec given on the context **wins** over the storage's — that is the restricted reader above.

### Without either, the context does not build

A domain event declaring a `Shreddable` cannot be registered on a context with no codec. `build()` fails rather than store personal data in the clear with no key to destroy:

```text
java.lang.IllegalArgumentException: event type com.example.banking.BankingDomain$BankingDomainEvent$CustomerRegistered
declares a Shreddable component but this store has no ShreddingCodec configured; personal data would be stored
in the clear and could never be erased. Configure shredding on the storage builder, or via
EventStore.on(storage).shredding(codec).
```

The key store stays yours, like the storage: `terminate()` closes the store the context built and nothing else. Where the key store shares the storage's `DataSource`, closing the storage and then the `DataSource` is enough.

## Reading Protected Values

A `Shreddable` is in one of three states: **present**, **shredded** (its key was destroyed) or **withheld** (it exists, and this reader may not see it). `orElse`, `map`, `orElseGet` and `toOptional` read the present value; `isShredded()` and `isWithheld()` tell the other two apart. A live read model, projected from the events on every read:

```java
public class CustomerDetailsReadModel implements ReadModel<BankingDomainEvent> {

    public record CustomerDetails ( String name, String city ) { }

    private final CustomerId customerId;
    private CustomerDetails details;

    public CustomerDetailsReadModel ( CustomerId customerId ) {
        this.customerId = customerId;
    }

    @Override
    public EventQuery eventQuery ( ) {
        return EventQuery.forEvents(EventTypesFilter.of(CustomerRegistered.class), CUSTOMER.tags(customerId));
    }

    @Override
    public void when ( Event<BankingDomainEvent> event ) {
        if ( event.data() instanceof CustomerRegistered registered ) {
            details = new CustomerDetails(
                    shown(registered.name()),
                    shown(registered.address().map(Address::city)));
        }
    }

    /** Erased and withheld are different answers, and a screen should say which. */
    private static String shown ( Shreddable<String> value ) {
        if ( value.isShredded() ) return "[erased]";
        if ( value.isWithheld() ) return "[withheld]";
        return value.orElse(null);
    }

    public Optional<CustomerDetails> details ( ) {
        return Optional.ofNullable(details);
    }
}
```

Registered as every live read model is, `builder.readmodel(CustomerDetailsReadModel.class).live()`. Read through the full context it shows `Alice Martin, Ghent`; read through the names-only context above it shows `Alice Martin, [withheld]`.

A key store that cannot be reached throws `ShreddingException` rather than answering "erased": a projection that recorded an outage as an erasure would write that gap into its read model for good. Through a projector that surfaces as a failed run, retried with backoff like any transient failure (see [Running in Production](/posts/eventmodeling-running-in-production/#when-things-fail)).

## Erasing

### Who may erase: `PrivacyCapability`

Erasure is irreversible and answers a request from a person, under an authority neither a web controller nor an operations dashboard carries. It is therefore its own audience, `PrivacyCapability`, and the component handling erasure requests holds that type — not the whole context, and not the application surface, which has no `erase` at all (see [Who May Do What](/posts/eventmodeling-who-may-do-what/)):

```java
PrivacyCapability privacy = banking;   // a built context already is one
```

| method | erases | answers |
|---|---|---|
| `erase(subjectType, subjectId, reason)` | the **person**: every key held for that type and id, in every category | a `SubjectErasureReport` with one `ErasureReport` per category that held live keys |
| `eraseCategory(subject, reason)` | **one category**: the keys of the category the `DataSubject` names, and no other | an `ErasureReport` |

The plain name is the one a request to be forgotten calls for, and it takes no category, so it cannot be narrowed by accident. `eraseCategory` carries the qualifier because it leaves the person's other categories readable — right for "forget my old address", wrong for a request to be forgotten, where a category left behind is an erasure reported as done and not performed. Two parameter shapes keep the two apart at compile time.

The `ErasureReason` is recorded beside every destroyed key, and is the whole audit trail — the events record nothing about an erasure, since they are never rewritten. Name the request it answers.

### End to end

The scratch project behind these guides registers Alice, erases her address, then erases her:

```java
CustomerId alice = CUSTOMER.newId();
app.execute(new RegisterCustomerCommand(alice, "Alice Martin", new Address("Main Street 1", "Ghent")));
app.read(CustomerDetailsReadModel.class, alice).details();
// CustomerDetails[name=Alice Martin, city=Ghent]

ErasureReport addressOnly = privacy.eraseCategory(
        BankingDomain.customerSubject(alice).withCategory("address"),
        ErasureReason.of("customer moved, asked to forget the old address #311"));
// erased customer/5c22.../address: 1 key(s) shredded at 2026-10-12T09:54:12Z (customer moved, ...)
app.read(CustomerDetailsReadModel.class, alice).details();
// CustomerDetails[name=Alice Martin, city=[erased]]

SubjectErasureReport person = privacy.erase("customer", alice.value(), ErasureReason.of("GDPR art.17 request #4711"));
// person.keysShredded() == 1, person.categoriesErased() == [default]
app.read(CustomerDetailsReadModel.class, alice).details();
// CustomerDetails[name=[erased], city=[erased]]

privacy.erase("customer", alice.value(), ErasureReason.of("again")).isNoop();
// true
```

Note what the second report says: the whole-person erasure found live keys in `default` only, because `address` was already gone. Both erasures are **idempotent** — a subject with no live keys reports `isNoop()` rather than failing — so a request handler can safely retry. Data appended for the same person afterwards is sealed under a **fresh** key and reads normally; only what was sealed under the destroyed keys is gone.

## Read Models Are Not Erased

This is the part that needs a decision per application. An erasure reaches the event log at once, and a live read model — projected from the events on every read — shows it at once, as above. But an **eventually consistent** read model holds its own copy of whatever its projection wrote, and its bookmark means it never re-reads the events it already projected. The same run, with a directory of customer names kept as a `PublishingReadModel`:

```text
before erasure:  {5c22...=Alice Martin}
after erasure:   {5c22...=Alice Martin}
```

The log is compliant; that read model still holds the name, in full, until something removes it. Nothing in the framework notices an erasure on its own. Three ways to handle it, from cheapest to most work:

| approach | how | fits |
|---|---|---|
| **keep personal data out** | eventually consistent read models hold ids; screens read names through a live read model, as above | most applications: the erasure is complete the moment the keys are gone |
| **rebuild** | stop the read model's projector, drop its state, start it again (see [Running in Production](/posts/eventmodeling-running-in-production/#operating-a-running-context)); the replay reads the values as shredded | rare erasures, read models cheap to rebuild |
| **record the erasure as a fact** | a domain event that read models holding a copy act on | read models that must hold personal data, and must drop it promptly |

The third is a command and a slice like any other. Its event says that a customer was forgotten, and nothing about what they were:

```java
/** Records that a customer's personal data was erased, so read models holding a copy can drop it. */
public record ForgetCustomerCommand ( CustomerId customerId ) implements Command<BankingDomainEvent> {
    @Override
    public void execute ( CommandContext<BankingDomainEvent, BankingDomainEvent> context ) {
        context.noDecisionModels().raiseEvent(new CustomerForgotten(customerId), CUSTOMER.tags(customerId));
    }
}
```

The read model holding names drops its copy on it:

```java
@Override
protected Map<String, String> apply ( Map<String, String> names, Event<BankingDomainEvent> event ) {
    Map<String, String> next = new LinkedHashMap<>(names);
    switch ( event.data() ) {
        case CustomerRegistered registered ->
                next.put(registered.customerId().value(), registered.name().orElse("[erased]"));
        case CustomerForgotten forgotten ->
                next.remove(forgotten.customerId().value());   // drop our copy
        default -> { }
    }
    return Map.copyOf(next);
}
```

And the component handling erasure requests does both, holding the two audiences it needs and nothing more:

```java
public class ErasureRequests {

    private final PrivacyCapability privacy;
    private final BankingApi banking;

    public ErasureRequests ( PrivacyCapability privacy, BankingApi banking ) {
        this.privacy = privacy;
        this.banking = banking;
    }

    public SubjectErasureReport forget ( CustomerId customerId, String requestReference ) {
        // 1. destroy the keys: from here on the events cannot produce the data
        SubjectErasureReport report = privacy.erase("customer", customerId.value(),
                ErasureReason.of("GDPR art.17 request " + requestReference));
        // 2. tell the read models holding a copy to drop it
        banking.execute(new ForgetCustomerCommand(customerId), "forget/" + requestReference);
        return report;
    }
}
```

In the run above, the directory went from `{5c22...=Alice Martin}` to `{}` once the projector caught up.

**The order is the guarantee.** The key store and the event store cannot share a transaction, so one step can succeed while the other fails. Erasing first means the worst case is a read model still showing data the log can no longer produce — visible, and corrected by retrying the request or by a rebuild. The other order could leave "forgotten" in the log while the data is still readable, which nobody would notice. Both steps are safe to retry: the erasure is idempotent, and the command's idempotency key derives from the request, so a retried request records the fact once.

Give the forgetting command no "already forgotten" guard: refusing a second request would leave a stale copy in place after a key had genuinely gone. And note that a replay after the erasure needs no marker at all — the projection reads the destroyed values as shredded and never stores them.

### What else erasure does not reach

| where | why | what to do |
|---|---|---|
| outbound events a dispatcher already sent | they left the system | your integration's contract; prefer publishing ids over personal data |
| caches, search indexes, files | never consulted the key store | invalidate them as part of the request |
| another instance's key cache | the PostgreSQL key store caches resolved keys, one hour by default; an erasure on one instance is seen by the others within that | state the bound in your data protection notice, or configure a shorter time-to-live |

The eventstore guide's [What Erasure Does Not Reach](/posts/eventstore-erasing-personal-data/#what-erasure-does-not-reach) has the full list, and its [audit view](/posts/eventstore-erasing-personal-data/#auditing-what-is-protected-and-what-was-erased) answers which subjects hold protected data and which erasures happened — without being able to decrypt anything.

## Testing

In the published test bases shredding is always on: `AbstractBoundedContextTest` — the base of `CommandTest`, `LiveModelTest` and the others — wires a fresh in-memory key store into every test (under `@ForEachBackend`, the backend's own: the SQL table on PostgreSQL), so a domain event carrying personal data works with no setup. `kernel()` is the context under test, so an erasure is one call, and `shreddingKeyStore()` hands out the key store for asserting on it:

```java
class CustomerDetailsReadModelTest extends LiveModelTest<BankingDomainEvent, BankingInboundEvent, BankingOutboundEvent> {

    // domainEventType(), inboundEventType(), outboundEventType() ...

    @Override
    public Class<? extends ReadModel<BankingDomainEvent>> getLiveModelClass ( ) {
        return CustomerDetailsReadModel.class;
    }

    @Test
    void anErasedCustomerReadsAsErased ( ) {
        CustomerId alice = CUSTOMER.newId();
        DataSubject subject = BankingDomain.customerSubject(alice);

        var test = given().event(new CustomerRegistered(alice,
                        Shreddable.of("Alice Martin", subject),
                        Shreddable.of(new Address("Main Street 1", "Ghent"), subject.withCategory("address"))),
                CUSTOMER.tags(alice));

        kernel().erase("customer", alice.value(), ErasureReason.of("GDPR art.17 request #4711"));

        test.when(alice)
            .then()
            .liveModel(model -> ((CustomerDetailsReadModel) model).details().orElseThrow())
            .is(new CustomerDetails("[erased]", "[erased]"));
    }
}
```

Erase **after** the events are appended: an erasure before them destroys nothing, and what is appended afterwards gets a fresh key. Override `createShreddingKeyStore()` to stand in a key store that fails, and check that an outage is not mistaken for an erasure. See [Testing Feature Slices](/posts/eventmodeling-testing/).

## Summary

- Wrap personal data in `Shreddable<T>`, bound to a `DataSubject` whose id is not itself personal data; split it into categories by what you will erase or withhold separately.
- Configure `shredding(...)` once, on the context or on the storage, with a key store that lives as long as the events.
- Hand erasure requests a `PrivacyCapability`; `erase` forgets the person, `eraseCategory` one category.
- Decide per read model how its copy goes: keep personal data out, rebuild, or record the erasure as a fact — erasing first.

## Related

- [Erasing Personal Data](/posts/eventstore-erasing-personal-data/) — the mechanism, key stores, auditing and entitlement
- [Who May Do What](/posts/eventmodeling-who-may-do-what/) — the audiences, `PrivacyCapability` among them
- [Choosing a Read Model](/posts/eventmodeling-read-models/) — live and eventually consistent read models
- [Running in Production](/posts/eventmodeling-running-in-production/) — storage, key stores per deployment, rebuilding a read model
