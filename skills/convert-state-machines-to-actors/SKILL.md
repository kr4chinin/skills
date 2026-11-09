---
name: convert-state-machines-to-actors
description: Convert a database-backed implicit state machine to an actor, the one owner every writer goes through. Use when asked to put all writers of persisted state behind one owner, to fix a race or lost update across several writers found by find-implicit-state-machines, or to convert several such machines one at a time.
---

# Convert implicit state machines to actors

An actor here is the one owner of a process's persisted state: every writer
sends it an event instead of writing. The actor is a function rebuilt for each
event. No mailbox, timer or statechart instance stays in memory, because the
database is the authority, and that is what makes the actor hold across
replicas and restarts.

Input: findings from `find-implicit-state-machines`, a `path:line` or a
ticket. Keep the finder's label. Convert **Confirmed** findings; for
**Likely**, step 2 supplies the missing confirmation; convert a **Smell** only
when the user asks. For several findings, start with
[Several machines](#several-machines).

A single conditional write (`UPDATE ... WHERE status = :expected`, checking the
affected row count) is often the whole fix. Convert to an actor when one
conditional write cannot cover the writers, or when the user asks for one
owner; otherwise recommend the conditional write and stop. If the finding does
not already list the writers, decide this after step 3.

## 1. Check for existing work

Read recent history of the files involved and look for open branches or pull
requests on the same code.

Done when no landed or in-flight change covers the finding. If one does,
report it and stop.

## 2. Reproduce

Write a test that replays the failing event sequence from the finding. A
sequential defect (a missing transition, an unhandled result) is replayed in
order. A race runs its writers in two independent database sessions against
the engine production uses: same engine, major version, isolation level and
connection pooling mode. Repository mocks and in-memory fakes have no locks or
isolation, and writers that share a process lock (a mutex, a channel, an
in-memory queue) prove nothing about replicas.

Force the interleaving from the database where the code has no seam: hold the
row lock or the key from a second connection, so writer A blocks at a known
point while writer B finishes. On a store that never blocks, place the barrier
in the test, between each writer's read and its write. Hooks inside the code
belong to test code only. Assert the reason from the finding (the final state,
the error), so the test goes red for that reason and no other.

The step ends one of three ways:

- **Red** on repeated runs, for the stated reason: continue.
- **Green** after the sessions really interleaved on that engine, isolation
  level and pooling mode: stop and report that the defect did not reproduce,
  with what you tried. The finder's label stays; a Confirmed static trace is
  not downgraded by a missing red test.
- **Blocked**: the test has not spoken yet. The production engine is
  unavailable, the test database is another engine, the isolation level or
  pooling mode differs, or the harness wraps each test in one transaction so
  a second session never sees the rows. Fix the harness where you can (Django
  `TransactionTestCase` or `pytest.mark.django_db(transaction=True)`, a test
  database on the production engine) and run the step again; otherwise report
  the blocker and leave the design for later.

## 3. Inventory the writers

Find every code path that writes the state, not only the one in the finding.
Search by column or field name, table name, repository method and raw SQL
text. Look in request handlers, background workers and schedulers, message
consumers, CLI and admin tools, seed and fixture scripts, data migrations and
backfills, and generic "save the whole entity" paths. Inspect the database
too (triggers, cascades, grants), and find out whether another service or
repository writes the same table.

| Writer | `path:line` | Guard today (lock, version, none) | After: actor, bypass (why) or out of band |
| ------ | ----------- | --------------------------------- | ----------------------------------------- |

End the table with `writers outside this repository: unknown` unless the user
named them. A shared database user or an empty search here is no evidence
about another repository.

Done when every search hit is classified. A missed writer is how the race
comes back.

## 4. Design the actor

The contract:

1. **Machine.** States, events, guards and named effects, with no I/O, clock
   or randomness; the actor stamps the time onto the event once. A typed
   `(state, event) -> next state and effects | rejected` function is enough.
   Use a statechart library only if the project already has one, and read
   [statechart libraries](references/statechart-libraries.md) before using it.
   A delay is a persisted due time plus the existing scheduler sending an
   event, never a timer in memory.
2. **Actor.** The only entry point. For each event, read the state and every
   guard input, run the transition, and commit the accepted event's writes
   (new state, duplicate receipt, effects) as one atomic bundle that the
   database accepts only if what was read still holds. With a row lock, the
   locking statement is the read, in the same transaction as the write. On a
   store without interactive transactions, the guard is the conditional write
   itself: on a condition failure, reload and run the event again. A lock
   that outlives the transaction, such as a session lock or a lock attribute
   you write and clear, is no guard. A rejected event writes nothing, and a
   replay the current code answers with success still gets that success. Logs
   and metrics go after commit.
3. **Persistence.** Keeps every guarantee the write path has today: locks,
   version columns, unique constraints, idempotency receipts, outbox.

Before designing the guard, read [guarded writes](references/guarded-writes.md).
The design answers each item that applies: guard, guard inputs, current read,
no row yet, atomic bundle, contention, duplicates, identity, outcomes, logs.

Ask the user, with two to four options and a recommendation, before leaving
this contract or the current behaviour: a schema change (a state, version or
receipt column, an outbox table), a new error response in a public API, an
external call whose outcome can be unknown and has no idempotency key the
receiver stores, an automatic resend of such a call, a writer that cannot go
through the actor, a store that cannot commit the bundle atomically.

### External effects

A call to another system cannot join the database transaction. Decide two
things separately; they often combine:

- **Must the result move the state?** Persist an awaiting state first:
  `pending` with a request id and the stored parameters, whoever dispatches.
  The result arrives as a second event.
- **Can delivery wait?** Write the call to an outbox in the same transaction
  and let a relay send it. The outbox table is a schema change to ask about,
  its relay ships in the same change, and the old direct call is removed in
  the same cutover, so each effect has exactly one sender.

Contention retries cover only the database transaction that rolled back; a
call that already left the process is outside that loop. An unknown outcome
goes to a reconciler shipped in the same change: it asks the receiver about
the request id where the receiver supports a lookup, and otherwise leaves the
row `uncertain` for a person. Read
[external effects](references/external-effects.md) before designing either
path.

Done when each part of the contract names its table, guard and code location
in this codebase, every transition that calls another system names its
awaiting state and sender, and every departure has the user's answer.

## 5. Write the scenario table

Before changing code, write
`precondition | event sequence | expected state | expected effects and response | test`,
one planned test per row, in the test file or the plan. Cover each that
applies:

- the same event twice, a retry after an unknown outcome, a redelivery after
  the first attempt committed;
- two writers in both orders, in separate sessions;
- a stale result after a newer one, a cancel or teardown;
- a crash between an external call and recording its result, and between
  commit and relay delivery (kill the process where the harness allows it);
- a rolled-back transaction leaving the old state;
- concurrent first writes when no row exists yet;
- stored states the new machine does not know, such as rows mid-flight today.

Mark a row n/a only with a `path:line` showing the sequence cannot happen. The
design you are about to write does not count as such evidence.

Done when every applicable row names its test.

## 6. Implement

- Write the machine, the actor and the persistence. Move each writer in this
  repository that assigns the state at runtime to the actor, and delete its
  old write path so one route remains.
- Generic saves keep writing their other columns and stop writing the state
  column. A test shows that a stale state in a loaded entity does not land.
- Applied migrations stay as they are. Fixtures and seeds may insert the
  initial state; list them as bypasses. Any other bypass takes the same guard.
- Old binaries keep running during a rolling deploy. A new column stays
  nullable and outside the guard until they are gone; the first guard is a
  predicate the current writers already store, usually the status.
- Before the machine rejects a stored state, look at which states exist in
  the data, through a bounded query or sample the user agrees to, written
  down. A backfill or repair is the user's decision.

Done when every inventory row reads actor, bypass or out of band, and the
project builds.

## 7. Verify

- The test from step 2 is green, and the losing session received the engine's
  own decision: a lock wait, a conditional-check failure or a unique
  violation. A correct final row alone does not pass.
- Every scenario row is green, effects and responses included.
- Race tests pass on repeated runs, and the existing suite is green.
- A new search for writes to the state finds only the actor and listed
  bypasses.
- Every outside writer is named as converted, guarded or absent; otherwise the
  report says the race is not proven fixed for the unknown ones.

Done when all five hold. The race stays open while old binaries still write
the old way; the report calls it fixed only after they are drained. Then hand
commit, review and pull request to the project's usual workflow.

## Several machines

Write a plan next to the finding report, or where the project keeps plans,
with one row per actor: `actor | writers that move | key check`. Order by
dependency: actors sharing a write path go in sequence, independent ones in
any order. Ask once whether to plan all of them or a pilot, unless the user
already said.

Run steps 1 to 7 for one actor at a time, one change each, every one with its
own red test. Local loops (quotas, rate limits, backoff, worker start and
stop) stay with their owners unless they write shared persisted state.

The plan is done when every finding has a row or a stated reason it is not
converted. The work is done when every row has passed step 7 or carries its
reason.

## Report

Write in the user's language:

1. What was red and is now green, with the test name, the guard, and the
   engine error the losing session received.
2. The writer inventory, bypasses with reasons, and outside writers.
3. Scenario rows marked n/a, each with its `path:line`.
4. What is left: remaining actors, open decisions, the rollout window.
