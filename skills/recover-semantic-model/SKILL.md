---
name: recover-semantic-model
description: Recover a program's minimal semantic model (the state that changes what the user sees, which transitions commute, repeat safely or depend on order, and what each effect requires), then audit it for nondeterminism, unenforced invariants, one fact stored twice and work nobody observes. Use when asked to recover or audit a semantic model, to check what is idempotent or order-dependent, or to split a refactor into provably equivalent and "trust me" changes with a breaking input for each.
---

# Recover the semantic model

A program's semantic model is the smallest description that predicts what its
users observe: the values that matter, the transitions that change them, and
what must hold before each effect. Recover it from the code first, then point
every finding at a row of it; a finding that names no row reads like ordinary
code review and is as hard to check.

This skill runs the request below, keeping its order and wording:

```text
Recover my program's minimal semantic model.

1. State: every value that changes what the user sees
2. Transitions: what commutes, what's idempotent, what's order-dependent
3. Dependencies: what must be true before each effect fires
   Then find:
   → nondeterminism in awaits, retries and event order
   → invariants enforced by hope
   → one fact stored in two places, drifting
   → work that never reaches the screen

Last, split my refactors into "provably equivalent" and "trust me". For each
"trust me", give the smallest input that breaks it.
```

Stay read-only: don't edit code, write tests or create tickets unless asked.
By default build the model (1 to 3) and run the four passes; run the split
when the request mentions refactors or names a diff. Asked for one part, do
that part and the rows it needs.

## Scope and screen

Use the scope the user gave; if the repository holds several programs and
there is no hint, ask which one. Report what you modelled, what you skipped
and the revision you read (with a diff in scope, its head).

Name the screen first: the surface where a person, or a system acting for
them, observes the program. For a UI, what renders. For a service, its
responses, the messages it sends, the records other programs read, the money
it moves. For a library, what its public API returns, throws and calls back.

Keep the model minimal: one row per stored copy of a fact, not per variable.
On a large scope, build one model per lifecycle or owner (a job, a document,
a family of consumers, the read side), main user flows first, and list the
rest as not modelled.

## 1. State: every value that changes what the user sees

Stored fields, in-memory and client state, values held by another system (a
workflow engine, a queue, a cache) and configuration that selects behaviour.
Number the rows S1, S2 and so on; T and E rows likewise.

| ID | Value | Lives in | Writers | Readers on the way to the screen | Source, or copy of |
| -- | ----- | -------- | ------- | -------------------------------- | ------------------ |

- **Writers**: every path that writes the value, as `path:line` or a T row;
  outside the repository, `unknown` unless the user named them.
- **Copy of**: the row restates another row: a counter beside the rows it
  counts, a cached status, a field repeated in a message, a per-process
  limit.
- A value nothing reads on the way to the screen stays in the table, with
  `unobserved` as its readers.

A status with its own lifecycle is one row; reconstructing its reachable
states is a different audit (see [Related skills](#related-skills)).

## 2. Transitions: what commutes, what's idempotent, what's order-dependent

Every event that writes a state row: a user action, a request, a delivered
message, a timer, a retry, a recovery job. Classify from the code, citing
the line that makes it so.

| ID | Event | Writes | Guard | Applied twice | Against other transitions |
| -- | ----- | ------ | ----- | ------------- | ------------------------- |

- **Idempotent**: applied twice, it leaves the same state and fires its
  effects once. Name what makes it so (a guard on the current state, a
  stored key, a unique constraint, a set instead of an increment), per row
  where rows differ.
- **Commutes**: two transitions give the same state and effects in either
  order.
- **Order-dependent**: the result depends on which lands first. Name what
  decides today: a lock, a transaction, one consumer per key, a sequence
  number, a caller's `await`, or nothing.

Classify pairs only where both orders can happen: they touch the same rows
and nothing orders their triggers. Write `unknown` instead of guessing; it
is a coverage gap, not a finding.

## 3. Dependencies: what must be true before each effect fires

An effect is anything that leaves the program or reaches the screen: a
response, a rendered value, a message, an email, a charge, a call to another
system. One row per precondition (E1a, E1b); a guard that protects an effect
appears in both tables.

| ID | Effect | Fired by | Must be true before | Enforcer |
| -- | ------ | -------- | ------------------- | -------- |

The enforcer is one of:

- **here**: a check, constraint, transaction or type, with `path:line` and
  what it checks against; a check against a copy is only as good as the
  copy;
- **elsewhere**: the caller, the database, the broker or another service,
  named, with where the guarantee is stated, or `unverified`;
- **none found**, with where you searched.

## Then find

Each pass reads the tables. Its results are candidates until they pass
[Check each finding](#check-each-finding).

### Nondeterminism in awaits, retries and event order

For each queue, retry loop, webhook or scheduler, first state the delivery
semantics the code intends, with the line or configuration that shows them:
at most or at least once, ordered or not, retried without bound or parked
after a limit. Where nothing says, don't guess: it is a question for the
owner.

Nondeterminism is a candidate only where it meets a row that cannot absorb
it: a repeat (a retry, a redelivery, a double click) meets a cell that is
not idempotent; two unordered events meet an order-dependent pair; an
`await` separates an enforcer's check from the effect it protects.

### Invariants enforced by hope

Hope is a precondition whose enforcer is **none found**: it holds only by
convention or timing. Report it with the search you made. **Here** is not a
finding. **Elsewhere** is neither hope nor safe: record it as a contract,
with what the user sees if the guarantee changes.

### One fact stored in two places, drifting

For each "copy of" row, list every writer of both rows. They drift when a
writer changes one without the other in the same atomic step, when a retry
repeats only one write, or when their lifetimes differ (a restart, a second
replica, a cache expiry). Say which is the source, which the screen reads,
which guards an effect, and what the user sees when they disagree.

### Work that never reaches the screen

Start from `unobserved` rows and effects nothing reads: a value never read,
a result discarded by a later guard, a recovery job for a state nothing
produces, a message with no consumer, an error caught and dropped, paid work
repeated although its first result was kept. Say what it costs (time, money,
a held lock or slot); cost is its whole impact. No reader in this repository
may still mean a reader elsewhere: a contract question.

## Check each finding

Candidates fail in three ways: the code was misread, an enforcer elsewhere
already covered the case, or two candidates shared a root cause. Before
reporting one:

1. **Anchor it** to the S, T and E rows it violates.
2. **Write its smallest breaking input** (below). Unobserved work has none:
   give the search for readers and the cost. Nor has an invariant nothing
   breaks today: name the assumption and the change that would break it.
3. **Try to disprove it.** Read the control flow at each cited line
   literally, not by its shape: a `try` around `return send()` catches
   nothing from a promise it did not await. Then look along the writers and
   guards of the anchored rows for a check that already rejects a step. If
   it is in this program, drop the candidate and list it as ruled out with
   the `path:line`; if it is elsewhere, the candidate becomes a contract.
4. **Merge by root cause.** Inputs that break at the same row and step are
   one finding with several symptoms; link findings whose fixes interact.

### Smallest breaking input

A numbered event sequence from a named starting state to a wrong result on
the screen:

```text
start: import 7 is processing, 200 lines in two chunks, rows_done = 0
1. chunk 1 delivered: 100 rows inserted, rows_done = 100, committed  consumer.ts:52-58
2. process dies before the ack                                       consumer.ts:58-63
3. chunk 1 redelivered: inserts skipped, rows_done = 200             consumer.ts:52-58, queue.ts:12
4. rows_done reaches 200: status = done, email queued                consumer.ts:66-70
seen: "import complete" while the table holds 100 of 200 lines
expected: completion only after chunk 2
```

- Each step is something a test can drive (a call, a delivered message, a
  second session, a killed process, a clock advance, a stored row), so the
  sequence becomes the regression test almost unchanged. Prose ("under load
  a message may be processed twice") is not a breaking input.
- Smallest: removing any step removes the failure. One entity, and two
  actors unless the defect needs a third.
- Every step cites the `path:line` that does it. A comment, or a call into
  code you could not read, is not a citation: mark the step `unverified`
  with what it depends on, and report the count (`4/4 steps cited`).
- If you cannot build one, say so and name the assumption the code rests
  on. Do not invent one.

## Last, split the refactors into "provably equivalent" and "trust me"

This part takes a diff: a commit range, a branch, a pull request or a patch.
Given none, use the current branch against the default branch (the remote's,
or local `main` or `master`), uncommitted changes included, and say so. If
that is empty, skip the split and report it as not covered; do not search
history for refactors.

Compare the rows each change touches at the base and at the head. A change
described as altering behaviour is not a refactor; list it apart.

- **Provably equivalent**: no row changes for any input, and the argument
  fits in a line or two: a rename or move, an extraction that keeps the
  calls and their order, a reordering of steps the transition table shows
  commute. Passing tests are evidence, not the argument.
- **Trust me**: a row changes, or the argument needs an assumption: an
  `await` added, removed or moved, a shifted transaction boundary, error
  handling moved across a call, a changed default, predicate or
  serialisation, anything near retries or acknowledgement.

For each "trust me", give the smallest input that breaks it, with the old
result and the new. If you tried and found none, say `no breaking input
found` and name the assumption. One that breaks is a finding: rank it and
report it once, here.

## Report

Write in the user's language. Read [report layout](references/report.md)
before writing the first table and follow it: the model first; findings
ranked by what the user sees go wrong and grouped by lifecycle, one group
per batch to ship; each with its fix kind and what the fix touches; owner
decisions, contracts with other systems, and what was ruled out or not
covered in their own sections.

The report describes one revision. Before a finding is implemented, by you
or anyone else, replay its breaking input against the current code: an
earlier fix from the same report may already have closed it.

## Related skills

This skill is complete on its own. Use the two below only when they are
available in the session. When one is not, do not reproduce its method from
memory: finish this audit, list the hand-off as not covered and give the
install command.

- `find-implicit-state-machines`: when a state row is a lifecycle (a status,
  a phase, a set of flags) whose reachable states need reconstructing, and
  for its checks on queues, workers, leases and durable workflows. Keep this
  report's layout; add that skill's label and evidence method to each
  finding as extra columns, as it defines them.
  `npx skills add kr4chinin/skills --skill find-implicit-state-machines`
- `convert-state-machines-to-actors`: when a fix has to put several writers
  of one persisted state behind a single owner.
  `npx skills add kr4chinin/skills --skill convert-state-machines-to-actors`
