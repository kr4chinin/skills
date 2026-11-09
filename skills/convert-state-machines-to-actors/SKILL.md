---
name: convert-state-machines-to-actors
description: Convert an implicit state machine whose state lives in a database into an actor, the single owner every writer goes through, with a pure transition function and the database as the authority. Use after find-implicit-state-machines, or when asked to fix a race, lost update or stuck status in persisted state.
---

# Convert implicit state machines to actors

An actor here is the single owner of one process's persisted state. Every
writer (request handler, worker, CLI, seed script, raw SQL) sends it an event.
The actor decides with a pure transition function and writes inside one
transaction. The database stays the authority, so there is no long-lived
in-memory actor: it would not coordinate replicas or survive a restart.

Input: one finding (`path:line`, a `find-implicit-state-machines` report, a
ticket) or an umbrella with several. For several, start with
[Several machines](#several-machines).

## 1. Check the ground

Read recent history of the files involved and look for open branches or pull
requests on the same code. If the fix already landed, report that instead.

## 2. Reproduce

Write a test that replays the failing event sequence against the same
database engine production uses. Repository mocks and in-memory fakes have no
locks or isolation levels, so they pass on the bug.

Make the order deterministic: hold two transactions open, use barriers or
hooks between read and write. Run the test on current code and assert the
reason from the finding (the final state, the error), so it goes red for that
reason and no other.

Done when the test is red on current code for the stated reason. If it stays
green, stop and report the finding as unconfirmed, with what you tried.

## 3. Inventory the writers

Find every code path that writes the state, not only the one in the finding.
Search by column or field name, table name, repository method and raw SQL
text. Look in request handlers, background workers and schedulers, message
consumers, CLI and admin tools, seed and fixture scripts, data migrations and
backfills, database triggers, and generic "save the whole entity" paths.

| Writer | `path:line` | Lock or version today | After: actor or bypass (why) |
| ------ | ----------- | --------------------- | ---------------------------- |

Done when every search hit is classified. A missed writer is how the race
comes back.

## 4. Design the actor

Use this contract unless the code makes it impossible:

1. **Machine.** States, events, guards and named actions, with no I/O, clock
   or randomness. Use a statechart library if the project already has one or
   the process needs hierarchy, parallel regions or delays; otherwise a typed
   `(state, event) -> next state and effects | rejected` function is enough.
   With XState v5: `setup().createMachine`, then rebuild the snapshot with
   `machine.resolveState({ value, context })` and call the pure
   `transition(machine, snapshot, event)`.
2. **Actor.** The only entry point. For each event, in one transaction: take
   the lock, load the state, run the transition. A rejected event returns a
   typed rejection and writes nothing. An accepted one writes the new state
   and records its effects in an outbox in the same transaction. Logs and
   metrics go after commit.
3. **Repository port.** Keeps every guarantee the code has today: row or
   advisory locks, version columns, unique constraints, idempotency receipts,
   outbox.

Details that decide correctness:

- **Locking.** `SELECT ... FOR UPDATE` locks only rows that exist. When the
  first write creates the row, or the invariant spans rows ("one default per
  catalog"), lock a key instead: an advisory lock on the owner id, or a
  unique constraint plus retry.
- **Identity.** Normalize the entity key once at the boundary (case,
  whitespace, format) before locking and comparing. Two spellings of one id
  take two locks and fail equality checks against stored values.
- **Rejections.** Return them as typed results that callers map to their
  existing errors. Report a constraint violation as itself, not as some
  nearby domain error.
- **Logs.** One event per outcome (`rejected`, `committed`, `effect_sent`,
  `stale_result_rejected`), keyed by entity id, without personal data.

Ask the user only when leaving this contract: a schema change (a new state or
version column), an external call whose outcome can be unknown and has no
idempotency key, a writer that cannot go through the actor. Offer two to four
options with a recommendation.

### External effects

A call to another system cannot join the database transaction.

- Prefer the outbox: the intent commits with the state, delivery happens
  after commit, at least once, and the receiver deduplicates by key.
- When the result is needed before the state can move on, persist `pending`
  with a request id first, call with that id as the idempotency key, then
  record the result. After a crash between call and record, retry with the
  same key only if the receiver honours it; otherwise move to `uncertain` and
  let a person or an explicit command decide. A paid or user-visible
  operation is re-sent only by that decision.
- Workers holding a lease carry the lease token in every write. An ack whose
  token no longer owns the row is rejected and logged.
- A late result is compared with the version captured at dispatch and
  rejected if the state has moved on.

## 5. Write the scenario table

Before changing code, write
`precondition | event sequence | expected state or invariant | test`. Cover
each that applies:

- the same event twice, and a retry after an unknown outcome;
- two writers in both orders;
- a stale result after a newer one, a cancel or teardown;
- identity change A → B → A;
- a crash between an external call and the write;
- a rolled-back transaction leaving the old state;
- concurrent first writes when no row exists yet.

Mark a row n/a only with the invariant that rules it out.

## 6. Implement

Write the machine, the actor and the port. Move every writer from the
inventory to the actor and delete the old write paths, so one route remains.
Take the state fields out of generic save and update paths: a save that
rewrites the whole row is a lost update waiting for its turn. A listed bypass
(a bulk migration, say) still takes the same lock. Keep public API shapes
unless the fix needs a change.

## 7. Verify

- The test from step 2 is green, and so is every row of the scenario table.
- Race tests pass on repeated runs, not once.
- The existing suite is green.
- Search for writes to the state again: only the actor and listed bypasses
  remain.

Done when all four hold. Then hand commit, review and pull request to the
project's usual workflow.

## Several machines

Write a plan file first, one row per actor: `actor | writers that move |
key check`. Order by dependency: actors sharing a write path go in sequence,
independent ones in any order. Ask once whether to plan all of them or a
pilot, unless the user already said.

Run steps 1 to 7 for one actor at a time, one change each, every one with its
own red test. Local loops (quotas, rate limits, backoff, worker start and
stop) stay with their owners unless they write shared persisted state.

## Report

Write in the user's language:

1. What was red and is now green, with the test name.
2. The writer inventory and any bypasses with reasons.
3. Scenario rows marked n/a, with the invariant.
4. What is left: remaining actors, open decisions.
