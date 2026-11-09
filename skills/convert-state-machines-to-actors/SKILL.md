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
affected row count) is often the whole fix. Convert to an actor when the
writers are many or scattered, the transitions need one home, or the user asks
for it. Otherwise recommend the conditional write and stop.

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
point while writer B finishes. Hooks inside the code belong to test code only.
Assert the reason from the finding (the final state, the error), so the test
goes red for that reason and no other.

The step ends one of three ways:

- **Red** on repeated runs, for the stated reason: continue.
- **Green** after the sessions really interleaved: stop and report that the
  defect did not reproduce, with what you tried. The finder's label stays; a
  Confirmed static trace is not downgraded by a missing red test.
- **Blocked**: the production engine is unavailable, the default test database
  is another engine, or the harness wraps each test in one transaction (Django
  `TestCase`, rollback-per-test fixtures) so a second session never sees the
  rows. Report the blocker and leave the design for later.

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
named them or the database shows none.

Done when every search hit is classified. A missed writer is how the race
comes back, and an unknown outside writer means the race cannot be called
fixed.

## 4. Design the actor

The contract:

1. **Machine.** States, events, guards and named effects, with no I/O, clock
   or randomness; the actor stamps the time onto the event once. A typed
   `(state, event) -> next state and effects | rejected` function is enough.
   Use a statechart library only if the project already has one, and read
   [statechart libraries](references/statechart-libraries.md) before using it.
   A delay is a persisted due time plus the existing scheduler sending an
   event, never a timer in memory.
2. **Actor.** The only entry point. For each event, in one transaction or one
   atomic write: load the current state under a guard and run the transition.
   A rejected event writes nothing. An accepted one writes the new state and,
   when the transition has effects, handles them as in
   [External effects](#external-effects). Logs and metrics go after commit.
3. **Persistence.** Keeps every guarantee the write path has today: locks,
   version columns, unique constraints, idempotency receipts, outbox.

Details that decide correctness:

- **Guard the write.** The engine accepts the new state only if what the
  transition read still holds. Use what the store supports: a row lock with
  the row read by the locking statement itself (`SELECT ... FOR UPDATE`), a
  compare-and-set (`UPDATE ... WHERE version = :v`, zero rows means reload and
  retry or reject), a conditional write on stores without interactive
  transactions, or a unique key. A lock that outlives the transaction (session
  locks such as `pg_advisory_lock` or MySQL `GET_LOCK`, a lock row you write
  yourself) survives crashes and pooled connections; take transaction-scoped
  locks (`pg_advisory_xact_lock`) on the transaction's own connection.
- **Current read.** Load after the guard is taken. An entity loaded earlier,
  an ORM identity-map hit or a snapshot read still shows the old version.
- **Guard inputs.** Everything a guard reads is loaded under the same guard,
  and locked too when another writer can change it.
- **No row yet.** In PostgreSQL a row lock covers only rows that exist; in
  MySQL gap locks depend on the isolation level. For a first creation, or an
  invariant across rows ("one default per catalog"), guard the invariant's
  scope: a transaction-scoped lock on the owner key, or a unique key plus
  retry.
- **Contention.** Take several locks in one fixed order, such as sorted keys.
  A deadlock or serialization failure is a normal outcome: retry the whole
  event a bounded number of times. The retry is safe because the failed
  attempt wrote nothing.
- **Duplicates.** The same event can arrive twice: a redelivery, or a client
  retry after the first attempt committed. When the current code treats a
  replay as success, the replay still returns success and writes nothing.
  When events carry a key, record it with the state and return the recorded
  outcome.
- **Identity.** Normalize the entity key once at the boundary (case,
  whitespace, format) before locking and comparing. Two spellings of one id
  take two locks and fail equality checks against stored values.
- **Outcomes.** Keep `rejected` (do not retry), `contended` (retry) and
  `uncertain` (reconcile) distinct in the result, and let callers map them to
  their existing errors. Report a constraint violation as itself, not as some
  nearby domain error.
- **Logs.** One event per outcome (`rejected`, `committed`, `effect_sent`,
  `stale_result_rejected`), keyed by entity id, without personal data.

Ask the user, with two to four options and a recommendation, before leaving
this contract or the current behaviour: a schema change (a state, version or
receipt column, an outbox table), a new error response in a public API, an
external call whose outcome can be unknown and has no idempotency key, a
writer that cannot go through the actor, a store that offers none of the
guards above.

Done when each part of the contract names its table, guard and code location
in this codebase, and every departure has the user's answer.

### External effects

A call to another system cannot join the database transaction. Choose by
rule:

- **The caller needs the result, or the call moves money or is visible to a
  user.** Persist `pending` with a request id, call outside the transaction
  with that id as the idempotency key, and record the result as a second
  event. An unknown outcome becomes `uncertain`; the call is repeated only by
  a person's or an explicit command's decision.
- **The call can wait.** Write it to an outbox in the same transaction. The
  outbox table is a schema change to ask about, its consumer ships in the same
  change, and the current call stays until that consumer runs.

Read [external effects](references/external-effects.md) before designing
either: it covers relays, leases, late results and recovery.

## 5. Write the scenario table

Before changing code, write
`precondition | event sequence | expected state | expected effects and response | test`,
one planned test per row, in the test file or the plan. Cover each that
applies:

- the same event twice, a retry after an unknown outcome, a redelivery after
  the first attempt committed;
- two writers in both orders, in separate sessions;
- a stale result after a newer one, a cancel or teardown;
- a crash between an external call and recording its result (kill the process
  where the harness allows it);
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
- Before the machine rejects a stored state, query the distinct states in the
  data. A backfill or repair is the user's decision.

Done when every inventory row reads actor, bypass or out of band, and the
project builds.

## 7. Verify

- The test from step 2 is green, and it shows the database blocked or
  rejected the losing session.
- Every scenario row is green, effects and responses included.
- Race tests pass on repeated runs, and the existing suite is green.
- A new search for writes to the state finds only the actor and listed
  bypasses.

Done when all four hold. Old binaries keep writing the old way until a
rolling deploy drains them; say so in the report. Then hand commit, review and
pull request to the project's usual workflow.

## Several machines

Write a plan next to the finding report, or where the project keeps plans,
with one row per actor: `actor | writers that move | key check`. Order by
dependency: actors sharing a write path go in sequence, independent ones in
any order. Ask once whether to plan all of them or a pilot, unless the user
already said.

Run steps 1 to 7 for one actor at a time, one change each, every one with its
own red test. Local loops (quotas, rate limits, backoff, worker start and
stop) stay with their owners unless they write shared persisted state.

Done when every finding has a row or a stated reason it is not converted.

## Report

Write in the user's language:

1. What was red and is now green, with the test name and the guard the
   database evaluated.
2. The writer inventory, bypasses with reasons, and outside writers.
3. Scenario rows marked n/a, each with its `path:line`.
4. What is left: remaining actors, open decisions, the rollout window.
