# External effects

Read this when a transition calls another system. The skill's step 4 decides
whether the call needs an awaiting state, an outbox or both; this file covers
how each stays correct.

## Awaiting state (`pending`)

- Dispatch and recording are two actor events in two transactions. No
  transaction or lock stays open across the external call.
- Store the request parameters with the request id. A resend reuses the key
  only with the same parameters.
- A crash before the call and a crash after it leave the same `pending` row.
  Treat both as unknown until reconciled.
- The reconciler ships in the same change and owns stale `pending` rows. It
  first asks the receiver about the request id, where the receiver supports a
  lookup. It resends only when the receiver documents and stores that key
  (check its documentation; an id minted only in this service is not such a
  key) and the user allowed automatic resend for this call. Everything else
  stays `uncertain` for a person.

## Outbox

- Only the relay sends. The old direct call is removed in the same cutover
  that enables the relay; two senders deliver the effect twice.
- Key each effect by the event key plus a stable per-transition effect id or
  position. One event can emit the same named effect several times with
  different parameters, so the name alone collides.
- Claim in a short transaction: take rows with `SELECT ... FOR UPDATE SKIP
  LOCKED`, stamp the claim owner and an expiry, commit. Send outside any
  transaction, then mark sent on the condition that the claim is still held.
  Expired claims are claimable again, which is how a crashed relay's rows
  move on.
- When one entity's effects must stay in order, give them a per-entity
  sequence assigned in the actor's transaction and make only the earliest
  unsent effect of each entity claimable. `SKIP LOCKED` alone lets another
  relay skip an earlier row and send a later one first. When order does not
  matter, leave it out.
- Delivery is at least once. A receiver with no deduplication key (email,
  SMS, many webhooks) eventually gets a duplicate after a crash between send
  and mark-sent: make the effect safe to repeat, or give it an awaiting state.

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
