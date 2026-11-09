# Guarded writes

Read before designing the actor's guard; answer each heading that applies.

## Guard

The engine accepts the new state only if everything the transition read
still holds:

- **Row lock.** Read with the locking statement itself
  (`SELECT ... FOR UPDATE`) and write in the same transaction.
- **Compare-and-set.** `UPDATE ... SET state = :next, version = :v + 1 WHERE
  id = :id AND version = :v`; every writer and bypass bumps `version`. On zero
  rows, reload, check the receipt and rerun the transition; resending the old
  next state is an overwrite.
- **Conditional write** on a store without interactive transactions: the
  condition is the guard; on failure, reload and rerun. No lock attribute.
- **Unique key.** Guards creation and uniqueness claims only, never an
  update.

A lock that outlives the transaction is no guard. A session advisory lock
(`pg_advisory_lock`, MySQL `GET_LOCK`) leaks to the next user of a pooled
connection, and a lock row committed on its own survives a crash. In
PostgreSQL use `pg_advisory_xact_lock` (the two-integer form or a wide hash,
so unrelated keys do not contend). MySQL has no transaction-scoped named
lock: lock the owner row by unique key, inserting it first with an
ignore-conflict insert when it may not exist.

## Guard inputs and current read

Everything the guard reads is part of it; the entity's version does not cover
a parent or quota row. With locks, load after taking them (for a row lock the
locking read is the load) and lock every input another writer can change.
With compare-and-set or a conditional write, read first and put each input's
version or status into the write's condition or transaction, or fold it into
the guarded row; on failure, reload from the database. An entity loaded
earlier, an ORM identity-map hit or a snapshot read shows the old version.

## No row yet

A PostgreSQL row lock covers only existing rows. In MySQL what a locking read
locks depends on the index, predicate and isolation level, so lock by
unique-key equality. For a first creation or a cross-row invariant ("one
default per catalog"), guard its scope: the owner row, a transaction-scoped
lock on the owner key, or a unique key plus retry for a uniqueness claim. An
aggregate invariant ("allocations within quota") needs the scope row locked.

## Atomic bundle

The state with its full context, the receipt and the `pending` intent or
outbox rows commit together. Without multi-item transactions that means one
item holding all of them, or the store's transactional API. If the bundle
cannot be atomic, ask the user.

## Contention

- Take several locks in one fixed order, such as sorted keys.
- Deadlocks and serialization failures are normal. Roll back the whole
  transaction (some engines roll back only the statement on a lock-wait
  timeout), then retry a bounded number of times with jittered backoff:
  reload, check the receipt, rerun.
- Retry only failures known to have rolled back; an unknown commit is
  resolved through the event key. Exhausted retries return `contended`.

## Duplicates

- Each entry point gives events a key (message or delivery id, client request
  id), or states that it re-evaluates duplicates against the current state.
  That works only when a replay is a no-op ("mark paid"); an increment or
  debit needs a key, and without one an unknown commit is `uncertain`, not
  retried.
- Keep one receipt per key, a unique row keyed by entity and event key, in
  the same bundle and for as long as redelivery is possible. A "last event"
  column cannot answer an older replay.
- After taking the guard and before the transition, return the recorded
  outcome for a known key. The same key with different parameters is a
  conflict.

## Identity

Normalize the entity key once at the boundary (case, whitespace, format),
consistent with the collation, before locking or comparing.

## Outcomes and logs

Keep `rejected` (do not retry), `contended` (retry) and `uncertain`
(reconcile) distinct; callers map them to their existing errors. Report a
constraint violation as itself. Log one event per outcome (`rejected`,
`committed`, `effect_sent`, `stale_result_rejected`), keyed by entity id,
without personal data.
