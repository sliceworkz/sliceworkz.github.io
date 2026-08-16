---
layout: post
toc: true
title: Erasing Personal Data
description: Crypto-shredding — protecting personal data with Shreddable values, and erasing it by destroying keys rather than rewriting events
date: 2026-08-16 01:00:00
categories: [Eventstore Documentation,Eventstore API]
tags: [gdpr,crypto-shredding,personal data,erasure,shreddable]
---

This guide covers how the EventStore holds personal data in event payloads, and how it erases that data on request — by destroying an encryption key, never by touching a stored event.

The types you program against — `Shreddable`, `DataSubject`, `ErasureReason`, the two seams and the audit view — live in `org.sliceworkz.eventstore.shredding` in the **api** module, so no extra dependency is needed. The shipped key stores live with the backend they belong to: `org.sliceworkz.eventstore.infra.inmem.shredding`, `…infra.inmem.fs.shredding` and `…infra.postgres.shredding`.

## The Conflict, and the Way Out

Events are immutable historical facts. GDPR Article 17 gives an individual the right to have their personal data erased. Those two do not obviously coexist, and the naive reconciliations all fail:

- **Deleting the event** breaks the log's ordering guarantees, strands bookmarks that name it, and destroys the non-personal facts (an amount, a decision, a correlation) along with the personal ones.
- **Overwriting the personal fields in place** looks like it works and does not. On PostgreSQL an `UPDATE` writes a *new* tuple; the old one lives on until `VACUUM`, and before that it has already reached the write-ahead log, every replica and last night's backup. Nothing there is erased, and now the physical order no longer matches `event_position`, which is what the BRIN index assumes.
- **Nulling a field** turns any event whose record validates its own components into a *poison event*: it can never be constructed again, so every query and every projection over its stream fails permanently.

**Crypto-shredding takes the other route.** Personal data is encrypted, in place, under a key held for the person it belongs to. Erasure destroys the key. The event is never written to again, so it stays byte-identical wherever it has already been copied — and every one of those copies becomes unreadable at the same instant, with nothing to chase.

```java
eventStore.erase(DataSubject.of("customer", "alice-42"),
                 ErasureReason.of("GDPR art.17 request #4711"));
```

That call writes **nothing** to the events table.

## Declaring Personal Data: `Shreddable<T>`

A record component holding personal data is declared `Shreddable<T>`:

```java
public sealed interface PaymentEvent {

    record PartyDetails(String name, String iban) { }

    record TransferMade(
            String transferId,
            Money  amount,
            String fromCustomerId,                 // pseudonymous — survives erasure
            String toCustomerId,                   // pseudonymous — survives erasure
            Shreddable<PartyDetails> from,         // Alice's personal data
            Shreddable<PartyDetails> to            // Bob's personal data
    ) implements PaymentEvent { }
}
```

The wrapper does three things at once that no annotation on a plain field can:

- **It declares in the type system that this component is personal data.** The compiler, not a convention, is what makes that visible at every use site.
- **It binds that data to a data subject**, so the store knows *whose* it is and can therefore erase it.
- **It makes "erased" a state the component can actually be in.** A shredded value is never `null`, so a record with a validating compact constructor still builds after its data is gone.

`Shreddable<T>` is a sealed interface with exactly two implementations:

| | Meaning | Components |
|---|---|---|
| `Shreddable.Present<T>` | the key still exists; the value is readable | `T value`, `DataSubject subject` |
| `Shreddable.Shredded<T>` | the key was destroyed; the value never reads again | `DataSubject subject`, `KeyId key` |

## Appending

The caller names whose data it is. The store mints a key per data subject on first sight, seals each value under the key for *its own* subject, and tags the event with the key ids it used.

```java
DataSubject alice = DataSubject.of("customer", "alice-42");
DataSubject bob   = DataSubject.of("customer", "bob-77");

payments.append(AppendCriteria.none(), Event.of(
        new TransferMade("t-9001", Money.eur("250.00"), "alice-42", "bob-77",
                         Shreddable.of(new PartyDetails("Alice Martin", "BE68 5390 0754 7034"), alice),
                         Shreddable.of(new PartyDetails("Bob Jansen",   "NL91 ABNA 0417 1643 00"), bob)),
        Tags.of(Tag.of("customer", "alice-42"), Tag.of("customer", "bob-77"))));
```

`Shreddable.of(value, subject)` encrypts nothing itself — sealing happens on append, when the store resolves the subject to a key. Two values for the same subject in one event share one key; values for different subjects do not.

Both the value and the subject are rejected if null. A `Shreddable` holding `null` would be indistinguishable from one that had been erased, and a `Shreddable` with no subject is data the store cannot attribute and therefore cannot erase.

## Reading

The common path needs no ceremony:

```java
String payer = transfer.from().map(PartyDetails::name).orElse("[erased]");
```

`Shreddable<T>` carries the small API you would expect — `isPresent()`, `isShredded()`, `toOptional()`, `map(...)`, `orElse(...)`, `orElseGet(...)` — plus `subject()`, which is available **whether or not the value still reads**. That is what lets a projection render `customer alice-42 (erased)` without consulting the key store at all.

Where the erased case deserves its own rendering, the sealed hierarchy makes the compiler ask:

```java
String payer = switch ( transfer.from() ) {
    case Shreddable.Present<PartyDetails>(var party, var subject) -> party.name();
    case Shreddable.Shredded<PartyDetails>(var subject, var key)  -> "customer " + subject.id() + " (erased)";
};
```

## Erasing

`EventStore.erase(subject, reason)` destroys every key held for a data subject and reports what it destroyed:

```java
ErasureReport report = eventStore.erase(
        DataSubject.of("customer", "alice-42"),
        ErasureReason.of("GDPR art.17 erasure request #4711, approved by DPO 2026-08-16"));

report.keysShredded();   // 1
report.shreddedKeys();   // [k-7f2a91c4]
report.shreddedAt();     // 2026-08-16T14:22:07Z
report.isNoop();         // false
```

Everything else on the affected events keeps working:

```java
transfer.amount();   // EUR 250.00                              unchanged
transfer.from();     // Shredded[customer/alice-42/default, …]  gone
transfer.to();       // Present[PartyDetails[Bob Jansen, …]]    unaffected — Bob has his own key
```

Ledgers still reconcile, correlation still works, and the audit trail still holds, because the pseudonymous identifiers, the tags, the timestamps and the non-personal payload were never encrypted in the first place.

**`ErasureReason` is not optional and may not be blank.** Shredding leaves the events untouched, so the key store row is the *only* record that an erasure happened, when, and on whose authority. Article 17 makes the erasure the obligation; the accountability principle of Article 5(2) is what makes the record worth keeping. Write something a data protection officer could act on, not `"erased"`.

**Erasure is idempotent.** A subject holding no keys — never appended for, or erased already — reports `isNoop()` rather than failing.

**Data appended *after* an erasure is readable again.** The subject gets a fresh key; only what was sealed under the destroyed ones is gone. Erasing twice therefore destroys two keys, not one, which is why erasure matches on *every* key a subject has ever held rather than only the active one.

> **Erasure notifies nothing.** Read models, caches, search indexes and downstream systems that already copied the personal data keep their copies, and projections hold bookmarks so they will never re-read the affected events on their own. Re-projecting anything that materialised the erased data is your application's responsibility — see [What Erasure Does Not Reach](#what-erasure-does-not-reach).
{: .prompt-warning }

## Data Subjects and Categories

`DataSubject` is the unit of erasure: a `(type, id, category)` triple.

```java
DataSubject alice = DataSubject.of("customer", "alice-42");   // category defaults to "default"
```

All three parts are required and non-blank. Blank parts are rejected rather than normalised, because a key held for a blank subject silently pools unrelated people's data under one key — and shredding it would erase all of them.

### The Subject Id Must Not Itself Be Personal Data

The subject is stored **in the clear** inside the sealed envelope, and it keys the key store, so it survives erasure by construction. Use a customer number, an account id, a surrogate key. An email address or a national identity number as the subject id defeats the entire mechanism: the identifier that remains after shredding *is* the personal data.

This is the same discipline the library already asks for when tagging events. `Tag.of("customer", customerId)` is safe to store and index precisely because `customerId` is pseudonymous.

### Categories: Erasing Part of a Subject's Data

Keys are held per `(type, id, category)`, not per subject, so a subject's data can be erased in parts. "Erase the marketing data, retain the financial records for the statutory period" is an ordinary request, and one key per subject makes it impossible to honour — shredding would take the financial history with it.

```java
DataSubject marketing = DataSubject.of("customer", "alice-42").withCategory("marketing");
DataSubject financial = DataSubject.of("customer", "alice-42").withCategory("financial");

// erases the marketing data only; the financial history keeps decrypting
eventStore.erase(marketing, ErasureReason.of("GDPR art.17 request #4711"));
```

Most events need only `DataSubject.DEFAULT_CATEGORY` (`"default"`), which `DataSubject.of(type, id)` applies. Reach for a category when parts of a subject's data are governed by different retention rules.

## Configuring Shredding

Shredding is configured on the storage builder. Each backend has a key store appropriate to it:

```java
// PostgreSQL — keys in <prefix>shredding_keys, on the same DataSource as the events
EventStore store = PostgresEventStorage.newBuilder()
        .prefix("acme_")
        .shredding()
        .buildStore();

// in-memory
EventStore store = InMemoryEventStorage.newBuilder()
        .shredding(new InMemoryShreddingKeyStore())
        .buildStore();

// file-persisted in-memory
Path directory = Path.of("eventstore-data");
EventStore store = InMemoryFsEventStorage.newBuilder()
        .directory(directory)
        .shredding(new InMemoryFsShreddingKeyStore(directory))
        .buildStore();
```

Or directly on the factory, when you build the store yourself:

```java
ShreddingCodec codec = AesGcmShreddingCodec.over(new InMemoryShreddingKeyStore());
EventStore store = EventStoreFactory.get().eventStore(storage, registry, MeterOptions.defaults(), codec);
```

> **Configuration is required, and it fails fast.** Opening a stream whose registered event types declare a `Shreddable` component on a store with **no** shredding configured throws `IllegalArgumentException` at `getEventStream` — before anything is read or written. Personal data silently stored in the clear is not a failure mode worth having, so this is an error rather than a fallback.
{: .prompt-warning }

`erase(...)` on a store with no codec throws `UnsupportedOperationException`: there are no keys to destroy.

Pair a file-backed store with a file-backed key store, or the events outlive the keys and every protected value reads as erased after a restart.

## The Two Seams

The shipped default is `AesGcmShreddingCodec` — AES-256-GCM, a random 96-bit IV per value, with the envelope metadata bound as additional authenticated data — over a `ShreddingKeyStore`. There are two places to substitute your own implementation, and which one you pick decides whether key material ever enters your JVM.

| Seam | What you take over | Reach for it when |
|---|---|---|
| `ShreddingKeyStore` | *where keys live* — the shipped AES-256-GCM encryption stays | keys belong in Vault, a cloud KMS, or another database |
| `ShreddingCodec` | *encryption and key handling together* | key material must stay inside an HSM and never reach this JVM |

```java
PostgresEventStorage.newBuilder().shredding(myVaultKeyStore).buildStore();   // narrow seam
PostgresEventStorage.newBuilder().shredding(myHsmCodec).buildStore();        // outer seam
```

### The Key Store Contract

- **`keyFor(subject)` creates on first sight and returns the same key afterwards.** It is called once per distinct subject per append, so it must be cheap and safe under concurrency: two threads appending for one subject at the same moment must end up with one key, not two.
- **`resolve(keyId)` answers empty *only* for a destroyed key.** See below.
- **`shred(subject, reason)` is idempotent** and returns what it actually destroyed.
- **Key material is never resurrected.** Destroying a key means the bytes are gone — but keep the row, with the material nulled and the reason and timestamp stamped, so the erasure stays auditable and the key id keeps resolving to *shredded* rather than to *unknown*.

### Ordering, When the Key Store Is Not Transactional With the Events

The key stores shipped with a SQL backend write keys on the same `DataSource` as the events, so a key mint and the append that seals under it commit together. An external key store cannot do that, and then the order is the whole guarantee:

**Mint the key first, append second.** A crash between the two leaves an orphan key, which decrypts nothing and costs nothing. The other order leaves an event whose key was never persisted — a value that can never be read, indistinguishable from an erasure nobody asked for.

## Empty Means Erased; Unavailable Means Throw

This is the contract that matters most, and the one place an implementation can do real damage.

`ShreddingCodec.unseal` and `ShreddingKeyStore.resolve` return an empty `Optional` **only** when the key has genuinely been destroyed. Every other failure — an unreachable Vault, an expired token, a timeout, a permissions problem, a corrupt envelope, an unsupported algorithm — must throw `ShreddingException`.

Collapse the two and a five-minute key-store outage renders every protected value as erased. Projections are at-least-once and advance a bookmark past what they have handled, so they write those gaps into read models permanently and never revisit them. A transient blip becomes silent, irreversible data loss in every downstream copy.

Reported as an exception instead, the read fails loudly, the bookmark does not move, and the projection recovers by itself once the key store is back.

`ShreddingException` is on the **retryable** side of the library's split — the same distinction the store already draws between `EventStorageException` (retry with backoff) and `EventDeserializationException` (never worth retrying). The typed serde rethrows a `ShreddingException` unwrapped, precisely so that "retry later" does not arrive at your code as an `EventDeserializationException`, which means "never retry". See [Error Handling](/posts/eventstore-error-handling/).

## Finding the Events Under a Key

Every event carries one `dek:` tag per distinct key its payload was sealed under, so "every event holding data protected by this key" is an ordinary tag query on the existing index — no extra column, no table scan:

```java
ErasureReport report = eventStore.erase(alice, ErasureReason.of("art.17 request #4711"));

for ( KeyId key : report.shreddedKeys() ) {
    stream.query(EventQuery.forEvents(EventTypesFilter.any(), Tags.of(KeyId.TAG_KEY, key.value())))
          .forEach(…);
}
```

`ErasureReport` deliberately does **not** compute an event count for you: it would read every matching event, which is a surprising cost to bury inside a call that is otherwise a single key-store write.

The tags are left in place when the key is destroyed. A `dek:` tag naming a key that no longer exists is a useful tombstone — it says an erasure touched this event, without saying what it took.

> **Key ids must be random, never derived from the subject.** The `dek:` tag is stored and indexed, so a key id computed as `sha256(email)` — or any other deterministic function of the subject — is re-identifiable by dictionary attack over any small domain, and survives the shredding it is supposed to enable. Mint key ids randomly and keep the association to the data subject inside the key store, which is the thing erasure destroys.
{: .prompt-danger }

## Auditing What Is Protected and What Was Erased

Because the events record nothing about an erasure, the key store is the whole account of it. `ShreddingAudit` is how a console or a compliance report reads that account — and it is deliberately the *only* way.

```java
ShreddingAudit audit = eventStore.shreddingAudit().orElseThrow();

audit.totals();                                                // subjects with live keys, live keys, shredded keys
audit.keys(KeyAuditQuery.forSubject("customer", "alice-42"));   // one person, every category
audit.keys(KeyAuditQuery.all().onlyShredded());                 // the erasure log: what, when, on whose authority
audit.keys(KeyAuditQuery.all().withCategory("marketing"));      // one retention category across subjects
```

A `KeyRecord` reports the key id, the subject, when it was minted, and — for a destroyed key — when and why:

```java
for ( KeyRecord key : audit.keys(KeyAuditQuery.all().onlyShredded()) ) {
    LOGGER.info("{} erased at {} ({})", key.subject(), key.shreddedAt().orElseThrow(), key.reason().orElseThrow());
}
```

- **`KeyRecord` carries no key material, and no method here returns any.** That separation is the whole reason this is a second interface rather than another method on `ShreddingKeyStore`: a dashboard credential granted it can see *that* data is protected and *when* it was erased, and never *what* it was. The PostgreSQL implementation does not merely refrain from reading `key_material` — the column is absent from every statement it issues, so key bytes cannot reach a log or a heap dump through this path.
- **Every query is bounded, and there is no cursor.** `KeyAuditQuery` always carries a limit, `DEFAULT_LIMIT` being 500. A store running for years holds one row per subject per category and never prunes the shredded ones, and unlike an event query there is nothing to resume from — so an accidental full enumeration is not offered. Widen it explicitly with `withLimit(...)`.
- **`forSubject(type, null)` is legal** and narrows to a subject *type*; an id without a type is rejected, because it would match subjects of every type, which is never what is meant.
- **Which *events* hold data under a key is not answered here** — the key store has never seen an event. That is the `dek:` tag query above.
- **It is optional, like leases.** A key store fronting a KMS that cannot enumerate returns empty from `audit()` and callers do without. All three shipped key stores implement it.

## Rotation Only Ever Applies Forward

There is deliberately **no** way to rotate a live key and re-seal what it protects. Re-sealing means rewriting stored events, which is the one thing this design exists to avoid: events staying byte-identical is exactly what makes destroying a key reach every copy of them — write-ahead logs, replicas, backups — with nothing to chase. A re-seal would have to reach all of those too, and would not.

So:

- **A subject whose keys are shredded gets a fresh key** for anything appended afterwards, and everything sealed under the old key stays sealed under it for as long as that ciphertext exists.
- **What *can* change without rewriting anything is the algorithm.** `alg` is recorded on every sealed envelope rather than assumed globally, so one store can hold values sealed under several algorithms at once: new appends use a new one while old events keep decrypting under the one they were written with. For a long-lived log, that agility is what rotation is usually reached for anyway — and it is the real defence against harvest-now-decrypt-later.
- **A key-encrypting key can be rotated freely**, since that lives inside your own codec or key store and never touches the events. That is where a KMS's rotation story belongs.

A codec must therefore dispatch on `Sealed.alg()` rather than assume its own current choice, and must throw rather than guess when it meets an algorithm it does not implement.

### On Post-Quantum

Mostly it does not apply here. Shor's algorithm breaks asymmetric cryptography, and the shipped codec uses none — no key exchange, no signatures, no public key anywhere in the design. Grover's algorithm is a quadratic speedup against symmetric ciphers, leaving AES-256 at roughly 128 bits of effective security, which NIST treats as quantum-resistant.

Shredding is in fact a *stronger* position than encryption at rest generally is. The threat model is an attacker holding ciphertext recovered from a backup and **not** holding the key, because it was destroyed — and no amount of computation recovers a key that does not exist.

Post-quantum becomes a real question one layer out, inside an implementation that wraps data keys under a key-encrypting key with RSA-OAEP or ECIES, which is what several KMS products do. That is precisely the decision the `ShreddingCodec` seam leaves to the implementer.

## The Sealed Envelope

A protected value is stored as a JSON object in place of the value, inside the ordinary payload document:

```json
"from": { "alg": "A256GCM",
          "dek": "k-7f2a91c4",
          "sub": { "type": "customer", "id": "alice-42", "category": "default" },
          "iv":  "yQ3mR1…",
          "ct":  "8Kd2vRhT…" }
```

It carries everything needed to decrypt the value later except the key itself, and everything needed to describe it honestly once the key is gone. The subject sits in the clear alongside the ciphertext, which is safe only because a subject id is required to be pseudonymous — and it is what lets a shredded value still say *whose* data it was with no key-store lookup.

The IV must be unique per value under a given key, and must never be derived from the event's position: one key protects many events.

Because this is one Jackson serializer working on one document, a `Shreddable` **anywhere** in the payload is sealed and unsealed correctly — nested several records down, as a `List` element, as a `Map` value. There is no special handling to remember and no shape it does not cover.

**Raw mode does not decrypt**, deliberately. A wildcard stream, an export or an import sees the envelope exactly as stored, which is what lets `EventStoreImporter` copy events with no keys and no domain classes on the classpath.

## The PostgreSQL Key Store

`.shredding()` on the PostgreSQL builder puts keys in `<prefix>shredding_keys`, on the same `DataSource` as the events. The existing schema machinery creates and validates that table alongside the others — see [Preparing the Database Schema Manually via DDL](/posts/eventstore-configuring-postgresql-storage/#the-shredding-keys-table).

**Resolved keys are cached with a TTL, one hour by default** (`PostgresShreddingKeyStore.DEFAULT_CACHE_TTL`). Without a cache, replaying a stream costs a query per protected value; with an unbounded one, an erasure performed by *another* instance would never be noticed here.

An erasure performed by *this* instance drops its own entries immediately, so the TTL bounds only the cross-instance case — which makes it the outer edge of "erased" for a multi-instance deployment, and a number worth stating in a data protection notice rather than discovering. `Duration.ZERO` disables the cache and makes an erasure effective everywhere at once, at the cost of a query per protected value.

```java
PostgresShreddingKeyStore keys = new PostgresShreddingKeyStore(dataSource, "acme_", Duration.ofMinutes(5));
EventStore store = PostgresEventStorage.newBuilder().prefix("acme_").shredding(keys).buildStore();
```

A key that was never seen is deliberately **not** cached as absent, so a shredded key still costs one query per read rather than reporting stale data as readable.

> **Colocation is a threat-model decision, not a default to accept blindly.** Keys in the same database as the ciphertext means an attacker with the database has both. What crypto-shredding still buys, unconditionally, is that a *completed erasure* holds everywhere the ciphertext has already spread — old backups, WAL, replicas. Where the keys must also be out of reach of whoever holds the database, pass a key store backed by a KMS or an HSM instead.
{: .prompt-warning }

The file-backed key store makes the same trade more starkly: it keeps `keys.jsonl` next to the events it protects, rewriting the whole file through an atomic move on erasure so destroyed material actually leaves it. It does **not** promise the bytes are unrecoverable from the device — a rewrite leaves the old blocks in place on a copy-on-write filesystem, an SSD with wear levelling, or a snapshotted volume. It is meant for development and tests.

## What Erasure Does Not Reach

Shredding erases the personal data **in the event log**. It does not, and cannot, reach:

| Where | Why | What to do |
|---|---|---|
| Read models and projections | they hold their own copies, and their bookmarks mean they never re-read the affected events | re-project, or handle the erasure explicitly |
| Caches and search indexes | never consulted the key store | invalidate them yourself |
| Downstream systems | already received the data | your integration's problem, not the store's |
| A running projector's key cache | keys are cached with a TTL | see the TTL note above |

The practical pattern is to make the erasure itself a fact your system knows about: append a domain event recording that the right to be forgotten was exercised, and have your projections act on it to remove the data from their read models. The event log erasure and the read-model erasure are then two halves of one auditable operation.

## Testing

The published testing module gives every backend test a store with shredding configured, over that backend's *own* key store — the SQL table on PostgreSQL, the file-backed one on inmem-fs, in-memory otherwise:

```java
class MyShreddingTest extends AbstractEventStoreTest {

    @ForEachBackend
    void erasureRemovesTheData ( ) {
        EventStore store = eventStoreWithShredding();
        …
    }
}
```

`eventStoreWithShredding(ShreddingKeyStore)` takes a key store of your choosing — for asserting what an erasure recorded, or for standing in a key store that fails, to check that an outage is not reported as an erasure. A custom backend supplies its own by overriding `EventStoreBackend.shreddingKeyStore(EventStorage)`; the default is in-memory, which every backend can use.

The TCK's `ShreddableEventDataTest` pins the whole contract per backend: the two-subject erasure, collections, the validating record, category independence, idempotent erasure and the fresh key afterwards, the `dek:` tags, the audit view — and, load-bearing, that an unreachable key store throws instead of reporting the data as erased. See [Testing](/posts/eventstore-testing/).

## Generating a Data Register

Because personal data is declared in the type system rather than in a comment, the GDPR register of processing activities that lists what personal data you hold is Java reflection over your event classes — every `Shreddable<T>` component is, by construction, personal data:

```java
for ( RecordComponent component : CustomerRegistered.class.getRecordComponents() ) {
    if ( Shreddable.class.isAssignableFrom(component.getType()) ) {
        register.add(CustomerRegistered.class.getSimpleName(), component.getName());
    }
}
```

Wrap the component in your own annotation carrying the purpose and the retention rule, and the register writes itself:

```java
public record CustomerRegistered(
        String id,

        @PersonalData(purpose = "required for personal communication")
        Shreddable<String> name,

        @PersonalData(purpose = "required for sending transactional e-mails")
        Shreddable<String> email,

        @PersonalData(purpose = "sending physical mail")
        Shreddable<Address> address

) implements CustomerEvent { }
```

The annotation is documentation; the `Shreddable` is the mechanism. Nothing can be marked as personal data and then quietly fail to be erasable, because the two are the same declaration.

## Related

- [Defining Events](/posts/eventstore-defining-events/) — declaring domain events, and where `Shreddable` fits
- [Configuring PostgreSQL Storage](/posts/eventstore-configuring-postgresql-storage/#the-shredding-keys-table) — the key table, its indexes and its privileges
- [Importing Events Between Stores](/posts/eventstore-importing-events/) — sealed values move as ciphertext, and the keys do not move with them
- [Error Handling](/posts/eventstore-error-handling/) — where `ShreddingException` sits in the retry taxonomy
