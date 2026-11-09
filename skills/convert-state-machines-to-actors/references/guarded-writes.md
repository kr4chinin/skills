# Guarded writes

Read this before designing the actor's guard. Each heading is one item the
design must answer when it applies.

## Guard

The engine accepts the new state only if everything the transition read still
holds. Pick what the store supports:

- **Row lock.** Read the row with the locking statement itself
  (`SELECT ... FOR UPDATE`) and write in the same transaction.
- **Compare-and-set.** `UPDATE ... SET state = :next, version = :v + 1 WHERE
  id = :id AND version = :v`. Every writer and bypass increments `version`. On
  zero rows, reload, check the duplicate receipt and run the transition again;
  re-sending the old next state against the new version is an overwrite.
- **Conditional write** on a store without interactive transactions. The
  condition is the guard; on a condition failure, reload and run the event
  again. Write no lock attribute.
- **Unique key.** Guards first creation and uniqueness claims only. It does
  not guard an update of an existing row.

## Guard inputs

Everything a guard reads is part of the guard. With row locks, load those rows
under the same transaction and lock them when another writer can change them.
With compare-and-set or conditional writes, put their version or status into
the same write's condition or transaction, or fold them into the guarded row.
The entity's own version does not cover a parent row or a quota row.

## Current read

With a lock, load after taking it; for a row lock the locking read is the
load. With compare-and-set or a conditional write, the read comes first and
the write's condition checks that it still holds; on a failure, reload from
the database. Either way, an entity loaded earlier, an ORM identity-map hit
or a snapshot read still shows the old version.

## Locks that outlive the transaction

A session advisory lock (`pg_advisory_lock`, MySQL `GET_LOCK`) is released
only when its session ends. Under transaction pooling it can leak to the next
user of the pooled connection, and the transaction it was meant to guard may
run on another connection. A lock row committed on its own survives a crash
as an orphan. None of these is a guard.

- PostgreSQL: take `pg_advisory_xact_lock` inside the transaction, on its
  connection. Hashing a string key to the lock's integer makes unrelated keys
  contend; use the two-integer form or a wider hash.
- MySQL has no transaction-scoped named lock. Lock the owner row itself by
  primary or unique key inside the transaction, creating it first with an
  insert that ignores conflicts when it may not exist.

## No row yet

A row lock covers only rows that exist in PostgreSQL. In MySQL, what a locking
read locks depends on the index, the predicate and the isolation level; lock
by unique-key equality, not by scanning a status column. For a first
creation, or an invariant across rows ("one default per catalog"), guard the
invariant's scope: the owner row or a transaction-scoped lock on the owner
key, or a unique key plus retry for a uniqueness claim. An aggregate invariant
("allocations within quota") needs the scope row locked.

## Atomic bundle

The accepted event's writes commit together: state and full context, the
duplicate receipt, the `pending` intent or outbox rows. On a store without
multi-item transactions that means one item or document holding all of them,
or the store's transactional API. A conditional write of the state followed
by a separate receipt or outbox write is a different contract; if the bundle
cannot be atomic, ask the user.

## Contention

- Take several locks in one fixed order, such as sorted keys.
- A deadlock or serialization failure is a normal outcome. Roll the whole
  transaction back (some engines roll back only the statement on a lock-wait
  timeout), then retry a bounded number of times with jittered backoff:
  reload, check the duplicate receipt, run the transition again.
- Retry only failures known to have rolled back. A commit whose result is
  unknown is resolved through the event key, not retried blind.
- Exhausted retries return `contended`.

## Duplicates

- Every entry point gives an event a key (message or delivery id, client
  request id), or the design states that this entry point re-evaluates
  duplicates against the current state. Re-evaluation works only when a
  replay is a no-op against the new state ("mark paid"). An event that is not
  (an increment, a debit) needs a key; without one, an unknown commit is
  `uncertain`, not retried.
- Keep one receipt per key: a unique row or item keyed by entity and event
  key, written in the same bundle as the state, kept as long as redelivery is
  possible. One "last event" column is overwritten by the next event and
  cannot answer an older replay.
- After taking the guard and before running the transition, return the
  recorded outcome for a key already seen. The same key with different
  parameters is a conflict.
- When the current code answers a replay with success, the replay still gets
  that success.

## Identity

Normalize the entity key once at the boundary (case, whitespace, format)
before locking and comparing, consistent with the database collation. Two
spellings of one id take two locks and fail equality checks against stored
values.

## Outcomes

Keep `rejected` (do not retry), `contended` (retry) and `uncertain`
(reconcile) distinct in the result, and let callers map them to their
existing errors. Report a constraint violation as itself, not as some nearby
domain error.

## Logs

One event per outcome (`rejected`, `committed`, `effect_sent`,
`stale_result_rejected`), keyed by entity id, without personal data.
