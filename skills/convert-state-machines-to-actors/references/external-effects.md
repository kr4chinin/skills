# External effects

Read when a transition calls another system.

## Awaiting state (`pending`)

- Dispatch and recording are two actor events in two transactions; nothing
  stays open across the call.
- Store the request parameters with the request id; a resend reuses the key
  only with the same parameters.
- A crash before or after the call leaves the same `pending` row: unknown
  until reconciled.
- The reconciler ships in the same change and owns stale `pending` rows. It
  asks the receiver about the request id where it can, and resends only when
  the receiver documents that key as idempotent, applying the same key and
  parameters once, even concurrently, for at least the resend window (an id
  minted only here is not one), and the user allowed automatic resend.
  Otherwise the row stays `uncertain` for a person.

## Outbox

- Only the relay sends. The direct call goes in the same cutover, or the
  effect goes out twice.
- Key each effect by entity, event key and a stable per-transition effect id
  or position: entities can share an event key, and one event can emit the
  same effect twice.
- Claim in a short transaction (`SELECT ... FOR UPDATE SKIP LOCKED`, stamp
  owner and expiry, commit), send outside any transaction, and mark sent only
  while the claim is held. Expired claims become claimable again.
- For per-entity order, assign a sequence in the actor's transaction and make
  only each entity's earliest unsent effect claimable. A relay whose claim
  expired mid-send can still deliver late, so promise order only to a
  receiver that applies effects by sequence.
- Delivery is at least once. If the receiver cannot deduplicate (email, SMS,
  many webhooks), make the effect safe to repeat; otherwise an expired claim
  on it goes to the reconciler, not back to the queue, under the resend rules
  above.

## Leases

- A worker carries a fencing token that grows on every takeover. The write's
  guard checks the token and the lease expiry, by the database clock, in the
  same atomic write; until a takeover the token alone still matches after
  expiry.
- Fencing protects the row, not the worker's calls. They are fenced only
  where the receiver rejects a stale token atomically with applying the call;
  a request id that only deduplicates does not fence distinct calls.

## Late results

- Accept a result only while the state still awaits that request id; an
  unrelated change to the entity does not invalidate it.
- A rejected result whose effect already happened (money moved) goes to
  compensation or manual review, not only to a log line.
