# External effects

Read this when a transition calls another system. The choice between the
`pending` pattern and the outbox is made in the skill's step 4; this file
covers how each one stays correct.

## The `pending` pattern

- Dispatch and recording are two actor events in two transactions. No
  transaction or lock stays open across the external call.
- Store the request parameters with the request id. A retry reuses the key
  only with the same parameters.
- Someone owns stale `pending` rows: a sweeper or the existing scheduler.
  First reconcile by asking the receiver about the request id, when it
  supports a lookup. Otherwise retry with the same key only if the receiver
  honours it, and move the rest to `uncertain`.
- A crash before the call and a crash after it leave the same `pending` row.
  Treat both as unknown until reconciled.

## The outbox

- Only the relay sends. The actor never sends an outbox effect itself after
  commit, or effects are sent twice by design.
- Derive each effect's key from the event key and the effect name, so a
  replayed event produces the same key instead of a new one.
- Relay replicas claim rows so they do not send the same one twice
  (`SELECT ... FOR UPDATE SKIP LOCKED`, or a claim lease) and mark them sent
  in a transaction.
- Keep one entity's effects in commit order. Ids are assigned before commit,
  so a later id can become visible first; do not advance past an earlier
  unsent effect for the same entity.
- Delivery is at least once. A receiver with no deduplication key (email,
  SMS, many webhooks) eventually gets a duplicate after a crash between send
  and mark-sent: make the effect safe to repeat, or treat it like the
  `pending` pattern.

## Leases

- A worker holding a lease carries a fencing token that grows with every
  takeover. The token is part of the write's guard, checked in the same
  atomic write; a read-then-compare in the worker leaves a gap.
- Judge expiry by the database clock, not the worker's clock.
- Fencing protects the row, not an effect the stale worker already sent. Pass
  the token or request id to the receiver where it accepts one.

## Late results

- Accept a result only while the state still awaits that request id. An
  unrelated change to the entity does not invalidate it.
- A rejected result whose effect already happened (money moved, the order
  since cancelled) goes to compensation or manual review, not only to a log
  line.
