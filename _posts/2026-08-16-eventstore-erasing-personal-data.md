---
layout: post
toc: true
title: Erasing Personal Data
description: Crypto-shredding — protecting personal data with Shreddable values, and erasing it by destroying keys rather than rewriting events
date: 2026-08-16 01:00:00
categories: [Eventstore Documentation,Eventstore API]
tags: [gdpr,crypto-shredding,personal data,erasure,shreddable,entitlement]
---

This guide covers how the EventStore holds personal data in event payloads, how it erases that data on request — by destroying an encryption key, never by touching a stored event — and how a reader that may not see personal data reads the events anyway.

The types you program against — `Shreddable`, `DataSubject`, `ErasureReason`, the two seams and the audit view — live in `org.sliceworkz.eventstore.shredding` in the **api** module, so no extra dependency is needed. The shipped key stores live with the backend they belong to: `org.sliceworkz.eventstore.infra.inmem.shredding`, `…infra.inmem.fs.shredding` and `…infra.postgres.shredding`.

## The Conflict, and the Way Out

Events are immutable historical facts. GDPR Article 17 gives an individual the right to have their personal data erased. Those two do not obviously coexist, and the naive reconciliations all fail:

- **Deleting the event** breaks the log's ordering guarantees, strands bookmarks that name it, and destroys the non-personal facts (an amount, a decision, a correlation) along with the personal ones.
- **Overwriting the personal fields in place** looks like it works and does not. On PostgreSQL an `UPDATE` writes a *new* tuple; the old one lives on until `VACUUM`, and before that it has already reached the write-ahead log, every replica and last night's backup. Nothing there is erased, and now the physical order no longer matches `event_position`, which is what the BRIN index assumes.
- **Nulling a field** turns any event whose record validates its own components into a *poison event*: it can never be constructed again, so every query and every projection over its stream fails permanently.

**Crypto-shredding takes the other route.** Personal data is encrypted, in place, under a key held for the person it belongs to. Erasure destroys the key. The event is never written to again, so it stays byte-identical wherever it has already been copied — and every one of those copies becomes unreadable at the same instant, with nothing to chase.

```java
eventStore.erase("customer", "alice-42", ErasureReason.of("GDPR art.17 request #4711"));
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

`Shreddable<T>` is a sealed interface with exactly three implementations:

| | Meaning | Components |
|---|---|---|
| `Shreddable.Present<T>` | the key still exists; the value is readable | `T value`, `DataSubject subject` |
| `Shreddable.Shredded<T>` | the key was destroyed; the value never reads again | `DataSubject subject`, `KeyId key` |
| `Shreddable.Withheld<T>` | the value exists, and *this reader* may not read it | `DataSubject subject`, `KeyId key` |

A withheld value says nothing about erasure — a reader that may not decrypt a value cannot tell whether it was erased, and is not told. See [Readers That May Not See Personal Data](#readers-that-may-not-see-personal-data).

## Appending

The caller names whose data it is. The store mints a key per data subject on first sight, seals each value under the key for *its own* subject, and tags the event with the key ids it used.

```java
DataSubject alice = DataSubject.of("customer", "alice-42");
DataSubject bob   = DataSubject.of("customer", "bob-77");

payments.append(Event.of(
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

`Shreddable<T>` carries the small API you would expect — `isPresent()`, `isShredded()`, `isWithheld()`, `toOptional()`, `map(...)`, `orElse(...)`, `orElseGet(...)` — plus `subject()`, which is available **whether or not the value still reads**. That is what lets a projection render `customer alice-42 (erased)` without consulting the key store at all. Note that an `orElse("[erased]")` fallback covers a *withheld* value too; where a reader can be denied, render the two apart.

Where the erased and withheld cases deserve their own rendering, the sealed hierarchy makes the compiler ask:

```java
String payer = switch ( transfer.from() ) {
    case Shreddable.Present<PartyDetails>(var party, var subject) -> party.name();
    case Shreddable.Shredded<PartyDetails>(var subject, var key)  -> "customer " + subject.id() + " (erased)";
    case Shreddable.Withheld<PartyDetails>(var subject, var key)  -> "customer " + subject.id();
};
```

## Erasing

`EventStore.erase(type, id, reason)` erases a **person**: it destroys every key held for that data subject, under every category their data was ever written under, and reports what it destroyed:

```java
SubjectErasureReport report = eventStore.erase(
        "customer", "alice-42",
        ErasureReason.of("GDPR art.17 erasure request #4711, approved by DPO 2026-08-16"));

report.keysShredded();       // 2
report.shreddedKeys();       // [k-7f2a91c4, k-19be03aa]
report.categoriesErased();   // [default, marketing]
report.categories();         // one ErasureReport per category that held live keys, each with its shreddedAt
report.isNoop();             // false
```

That is the erasure to reach for by default. An art.17 request names a person, not a retention category, and whoever answers it should not have to know which categories the person's data was ever written under. It takes a type and an id and no category, so it cannot be narrowed by accident. Erasing one category only is a [separate call](#categories-erasing-part-of-a-subjects-data).

Everything else on the affected events keeps working:

```java
transfer.amount();   // EUR 250.00                              unchanged
transfer.from();     // Shredded[customer/alice-42/default, …]  gone
transfer.to();       // Present[PartyDetails[Bob Jansen, …]]    unaffected — Bob has his own key
```

Ledgers still reconcile, correlation still works, and the audit trail still holds, because the pseudonymous identifiers, the tags, the timestamps and the non-personal payload were never encrypted in the first place.

**`ErasureReason` is not optional and may not be blank.** Shredding leaves the events untouched, so the key store row is the *only* record that an erasure happened, when, and on whose authority. Article 17 makes the erasure the obligation; the accountability principle of Article 5(2) is what makes the record worth keeping. Write something a data protection officer could act on, not `"erased"`.

**Erasure is idempotent.** A subject holding no keys — never appended for, or erased already — reports `isNoop()` rather than failing.

**Erasure is observed**, with the reason, when the store has an [observer](/posts/eventstore-observability-micrometer-prometheus-grafana/): an `Observation.Erase` that completes with the number of keys shredded and the categories erased.

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
ErasureReport report = eventStore.eraseCategory(marketing, ErasureReason.of("marketing opt-out #812"));
```

`eraseCategory(subject, reason)` destroys the keys of the category the `DataSubject` names, and no other — reporting success, because the erasure it names was performed. On a subject that also holds `financial` data it leaves that readable, which is right for "erase marketing, retain financial" and **wrong for an art.17 request**: that is `erase(type, id, reason)`. The two take different parameters precisely so that a call meaning one cannot silently become the other.

Most events need only `DataSubject.DEFAULT_CATEGORY` (`"default"`), which `DataSubject.of(type, id)` applies. Reach for a category when parts of a subject's data are governed by different retention rules — or are read by different services, since a category is the unit of [access](#readers-that-may-not-see-personal-data) as well as of erasure. Each category is one more key row per subject and one more key lookup per append that carries it: a handful per subject is the intended scale, not one per field. A category is forward-only: a value sealed under `default` cannot be re-categorised without re-sealing it, which means rewriting the event.

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

**The codec travels with the storage**, so `build()` honours `.shredding(...)` exactly as `buildStore()` does: a store built on that storage with `EventStore.on(storage).build()` seals and unseals with it. That matters on PostgreSQL in particular, where the no-argument `.shredding()` needs the `DataSource` the builder resolves. A store that must read the same storage differently gives the store builder a codec of its own, which wins:

```java
PostgresEventStorage storage = PostgresEventStorage.newBuilder().shredding().build();

EventStore store     = EventStore.on(storage).build();                                         // the storage's codec
EventStore reporting = EventStore.on(storage).shredding(ShreddingCodec.withholdingAll()).build();  // its own
```

The storage never seals or unseals anything itself, which is what keeps raw reads, exports and imports seeing the envelope as stored.

> **Configuration is required, and it fails fast.** Opening a stream whose registered event types declare a `Shreddable` component on a store with **no** shredding configured throws `IllegalArgumentException` at `getEventStream` — before anything is read or written. That check reads declarations, so it cannot see a `Shreddable` held behind a component declared as an interface; such a value fails the append instead, as an `EventSerializationException` with nothing stored. Personal data silently stored in the clear is not a failure mode worth having, so both are errors rather than fallbacks.
{: .prompt-warning }

`erase(...)` and `eraseCategory(...)` on a store with no codec throw `UnsupportedOperationException`: there are no keys to destroy.

Pair a file-backed store with a file-backed key store, or the events outlive the keys and, after a restart, every read of a protected value throws a `ShreddingException` naming a key the store never held.

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

- **`keyFor(subject)` creates on first sight and returns the same key afterwards.** It is called once per distinct subject per append, so it must be cheap and safe under concurrency: two threads appending for one subject at the same moment must end up with one key, not two. The key it returns must be durable before it is returned.
- **`resolveKey(keyId)` answers a sealed `KeyResolution`**: `Resolved(key)`, `Erased()` for a key that was destroyed, or `Withheld(reason)` for a key this caller may not have. Anything else — including a key id the store has never held — throws. See below.
- **`shred(subject, reason)` is idempotent** and returns what it actually destroyed. **`shredAllCategories(type, id, reason)`** is the whole-person erasure; its default throws `UnsupportedOperationException`, so a key store written before it is told rather than made to erase one category and report success.
- **Key material is never resurrected.** Destroying a key means the bytes are gone — but keep the row, with the material nulled and the reason and timestamp stamped, so the erasure stays auditable and the key id keeps resolving to *erased* rather than to *unknown*.

The outer seam mirrors it: `ShreddingCodec.open(sealed)` answers `Unsealed.Plaintext`, `Unsealed.Erased` or `Unsealed.Withheld`, and throws otherwise. The refusal carries the same name on the key store, the codec and the value, so one word follows a refused key from the key store to the reader.

### A Key Is Committed Before the Event Sealed Under It

`keyFor` runs while the payload is sealed, before the storage is handed anything to append — and the shipped PostgreSQL key store takes a connection of its own for it, even though it writes to the same `DataSource` as the events. No key store puts a key and its event in one transaction; **the order is the guarantee**: mint the key, durable before it is returned, then append.

A rolled-back or crashed append leaves an orphan key, which decrypts nothing and which the subject's next append seals under. The other order would leave an event whose key was never persisted — a value that can never be read, indistinguishable from an erasure nobody asked for. That cannot happen.

## Erased Means Erased; Unavailable Means Throw

This is the contract that matters most, and the one place an implementation can do real damage.

`ShreddingKeyStore.resolveKey` answers `Erased` and `ShreddingCodec.open` answers `Unsealed.Erased` **only** when the key has genuinely been destroyed, and `Withheld` only for a deliberate refusal to this caller. Every other failure — an unreachable Vault, an expired token, a timeout, a corrupt envelope, an unsupported algorithm, **a key id the store has never held** — must throw `ShreddingException`.

The last one deserves its own sentence. A shredded key keeps its row, so every shipped key store can tell "destroyed" from "never seen", and the second means the store is not the one the events were sealed against: the file-backed store pointed at the wrong directory, the PostgreSQL one at the wrong prefix or database, events imported without their keys. Reported as erased, that is the outage failure below applied to the *whole* store at once. The cost is that shredded rows must stay: pruning one turns that subject's events from "erased" into unreadable, with an error naming the key.

Collapse the two and a five-minute key-store outage renders every protected value as erased. Projections are at-least-once and advance a bookmark past what they have handled, so they write those gaps into read models permanently and never revisit them. A transient blip becomes silent, irreversible data loss in every downstream copy.

Reported as an exception instead, the read fails loudly, the bookmark does not move, and the projection recovers by itself once the key store is back.

`ShreddingException` is on the **retryable** side of the library's split — the same distinction the store already draws between `EventStorageException` (retry with backoff) and `EventDeserializationException` (never worth retrying). The typed serde rethrows a `ShreddingException` unwrapped, precisely so that "retry later" does not arrive at your code as an `EventDeserializationException`, which means "never retry". See [Error Handling](/posts/eventstore-error-handling/).

## Finding the Events Under a Key

Every event carries one `dek:` tag per distinct key its payload was sealed under, so "every event holding data protected by this key" is an ordinary tag query on the existing index — no extra column, no table scan:

```java
SubjectErasureReport report = eventStore.erase("customer", "alice-42", ErasureReason.of("art.17 request #4711"));

for ( KeyId key : report.shreddedKeys() ) {
    stream.query(EventQuery.forTags(Tags.of(KeyId.TAG_KEY, key.value())))
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
audit.categories();                                             // which categories exist, and how much under each
audit.keys(KeyAuditQuery.forKeys(keysOnAnEvent));               // are the keys this event carries still live?
```

A `KeyRecord` reports the key id, the subject, when it was minted, and — for a destroyed key — when and why:

```java
for ( KeyRecord key : audit.keys(KeyAuditQuery.all().onlyShredded()) ) {
    LOGGER.info("{} erased at {} ({})", key.subject(), key.shreddedAt().orElseThrow(), key.reason().orElseThrow());
}
```

- **`KeyRecord` carries no key material, and no method here returns any.** That separation is the whole reason this is a second interface rather than another method on `ShreddingKeyStore`: a dashboard credential granted it can see *that* data is protected and *when* it was erased, and never *what* it was. The PostgreSQL implementation does not merely refrain from reading `key_material` — the column is absent from every statement it issues, predicates included ("shredded" is judged by `shredded_at`), so key bytes cannot reach a log or a heap dump through this path, and the audit works for a [reporting role](/posts/eventstore-configuring-postgresql-storage/#a-role-that-must-not-read-personal-data) granted every column but that one.
- **Every query is bounded, and there is no cursor.** `KeyAuditQuery` always carries a limit, `DEFAULT_LIMIT` being 500. A store running for years holds one row per subject per category and never prunes the shredded ones, and unlike an event query there is nothing to resume from — so an accidental full enumeration is not offered. Widen it explicitly with `withLimit(...)`.
- **`categories()` is the inventory**: which categories of personal data the store holds, and how much under each (`CategoryTotals`: live subjects, live keys, shredded keys per category, most live subjects first). A category is the unit of erasure *and* of access, and only the key store knows which exist — so this is what an operator reads before deciding which categories a service is [restricted to](#readers-that-may-not-see-personal-data). An erased category stays listed, with zero live keys. Deriving it from `keys()` would be wrong on any store holding more keys than the query's limit, which is why it is a method rather than a recipe.
- **`KeyAuditQuery.forKeys(Set<KeyId>)` is the join back from an event.** An event carries its keys as `dek:` tags and in each envelope, and nothing else; whether those keys still exist is the key store's to say. A dashboard rendering an event asks for exactly those keys and can tell "protected" from "erased on … because …" without holding a key of its own. On PostgreSQL it is a primary-key lookup; the limit defaults to the number of keys asked for, an empty set is refused, and a key the store never held simply answers nothing.
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

### What the Shipped Codec Refuses When Sealing

`AesGcmShreddingCodec` measures the key it is handed. The JCE encrypts under a 128-, 192- or 256-bit AES key alike, so a key store minting the wrong length would otherwise seal without complaint under an envelope recording `A256GCM`. `seal` refuses a key that is not 256-bit AES material — and one whose material it cannot see at all, an HSM-resident key, which belongs behind a `ShreddingCodec` of its own — with a `ShreddingException` naming the key, nothing sealed. `open` deliberately does not measure: what is sealed is sealed.

The metadata GCM authenticates is the algorithm, key id, subject type, id and category joined with `|`, so `seal` also refuses a `|` in any of those fields — two labels differing only in where the `|` falls would otherwise authenticate as one. Keep subject types, ids and categories free of `|` with this codec.

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

**The cache is bounded in size too**: `DEFAULT_MAX_CACHED_KEYS`, 10.000 entries, least recently used evicted first. The TTL bounds how stale an entry can be, not how many there are, so a process resolving a key per subject over its lifetime — a projection replaying a stream of a million subjects — would otherwise hold every one of them for good. Below the bound nothing changes; above it, a key outside the working set costs one query when it comes round again. The four-argument constructor, `new PostgresShreddingKeyStore(dataSource, prefix, ttl, maxCachedKeys)`, sets the bound for a working set that is genuinely larger.

> **Colocation is a threat-model decision, not a default to accept blindly.** Keys in the same database as the ciphertext means an attacker with the database has both. What crypto-shredding still buys, unconditionally, is that a *completed erasure* holds everywhere the ciphertext has already spread — old backups, WAL, replicas. Where the keys must also be out of reach of whoever holds the database, pass a key store backed by a KMS or an HSM instead.
{: .prompt-warning }

The file-backed key store makes the same trade more starkly: it keeps `keys.jsonl` next to the events it protects, rewriting the whole file through an atomic move on erasure so destroyed material actually leaves it. It does **not** promise the bytes are unrecoverable from the device — a rewrite leaves the old blocks in place on a copy-on-write filesystem, an SSD with wear levelling, or a snapshotted volume. It is meant for development and tests.

## Readers That May Not See Personal Data

Not every reader may read everything. Access to a protected value *is* the ability to resolve its key, so who may read what is decided on the key seams, never in the read path — and a reader that may not gets the third state, `Shreddable.Withheld`.

**Why a third state rather than either existing answer.** Reported as `Shredded`, a projection renders "erased" for data that is not, and writes that into its read model for good. Reported as a `ShreddingException`, it means "retry later", so a projector that is merely not entitled fails its batch and never advances. Withheld is neither: the read completes, the projection decides how to render the gap, and everything the reader *is* entitled to still gets projected.

Three ways to limit a reader, from cheap to hard:

```java
// A reporting service: typed events, none of the personal data. It still needs a codec --
// registering a type that declares a Shreddable fails on a store with none
PostgresEventStorage.newBuilder().shredding(ShreddingCodec.withholdingAll()).buildStore();

// A service that reads names and never addresses: an in-process policy on the category
PostgresEventStorage.newBuilder()
    .shredding(AesGcmShreddingCodec.over(keyStore).restrictedTo(Set.of("identity")))
    .buildStore();

// The hard boundary: a key store that refuses keys this role is not granted
public KeyResolution resolveKey(KeyId key) {
    ...
    return new KeyResolution.Withheld("vault: 403");
}
```

- **`withholdingAll()`** withholds every value without touching a key store.
- **`restrictedTo(categories)`** decides on the category the envelope carries in the clear, before any key lookup, so a denied category costs no key-store traffic. It is symmetric — the codec seals nothing outside its categories either, and such an append fails as an `EventSerializationException` with nothing stored — and it passes erasure and the audit through *whole*, because an erasure that silently left another category readable while reporting success is the worst outcome an erasure can have. It is a data-minimisation boundary a deployment declares for itself, not a security boundary: the process still holds the codec.
- **The key store's refusal is the security boundary**: a KMS policy per service role, or on PostgreSQL a [role granted every column of the key table except `key_material`](/posts/eventstore-configuring-postgresql-storage/#a-role-that-must-not-read-personal-data), which `PostgresShreddingKeyStore` recognises by SQLSTATE `42501` and reports as `Withheld` (cached for the key TTL). The two compose.

**The unit of access is the unit of encryption: the `Shreddable` value, partitioned by category.** "Name but not address" is two wrapped values under two categories, chosen when the event is written — not one `Shreddable<ContactDetails>` holding both. Nothing inside one sealed value can be handed out on its own.

A withheld value **cannot be appended again**: this process never held the plaintext. And a withheld reader still sees the pseudonymous subject id, the category and the `dek:` tags — which is why the rule that subject ids and tags must not themselves be personal data carries the weight here.

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

`eventStoreWithShredding(ShreddingCodec)` takes a whole codec — a withholding or restricted one, for entitlement tests. `eventStoreWithShredding(ShreddingKeyStore)` takes a key store of your choosing — for asserting what an erasure recorded, or for standing in a key store that fails, to check that an outage is not reported as an erasure. A custom backend supplies its own by overriding `EventStoreBackend.shreddingKeyStore(EventStorage)`; the default is in-memory, which every backend can use.

The TCK's `ShreddableEventDataTest` pins the whole contract per backend: the two-subject erasure, collections, the validating record, category independence and the whole-person erasure across every category, idempotent erasure and the fresh key afterwards, the `dek:` tags, the audit view, reader entitlement — and, load-bearing, that an unreachable key store, or one asked for a key it never held, throws instead of reporting the data as erased. See [Testing](/posts/eventstore-testing/).

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
