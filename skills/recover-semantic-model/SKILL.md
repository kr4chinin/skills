---
name: recover-semantic-model
description: Recover a program's minimal semantic model (the state that changes what the user sees, which transitions commute, repeat safely or depend on order, and what each effect requires), then audit it for nondeterminism, unenforced invariants, one fact stored twice and work nobody observes. Use when asked to recover or audit a semantic model, to check what is idempotent or order-dependent, or to split a refactor into provably equivalent and "trust me" changes with a breaking input for each.
---

# Recover the semantic model

A program's semantic model is the smallest description that predicts what its
users observe: the values that matter, the transitions that change them, and
what must hold before each effect. Recover it from the code first, then point
every finding at a row of it. A finding that names no row reads like ordinary
code review and is as hard to check.

This skill runs the request below. The sections keep its order and wording
and add what each line needs to be checkable.

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

By default build the model (1 to 3) and run the four passes. Run the split
when the request mentions refactors or names a diff. Asked for one pass or
for the split alone, do that part and the rows it needs.

## Scope and screen

Use the scope the user gave. If the repository holds several programs and
there is no hint, ask which one. Report what you modelled, what you skipped
and the revision you read; with a diff in scope, model its head.

Name the screen first: the surface where a person, or a system acting for
them, observes this program. For a UI it is what renders. For a service it is
its responses, the messages and notifications it sends, the records and files
other programs read, the money it moves. For a library it is what its public
API returns, throws and calls back. Every later step asks whether something
reaches that surface.

Keep the model minimal: one row per stored copy of a fact, not per variable.
On a large scope, build one model per lifecycle or owner (one job, one
document, one family of consumers, the read side), starting with those the
main user flows pass through, and list the rest as not modelled.

## 1. State: every value that changes what the user sees

List each value whose change can change the screen: stored fields, in-memory
and client state, values held by another system (a workflow engine, a queue,
a cache) and configuration that selects behaviour. Number the rows S1, S2 and
so on; T and E rows likewise.

| ID | Value | Lives in | Writers | Readers on the way to the screen | Source, or copy of |
| -- | ----- | -------- | ------- | -------------------------------- | ------------------ |

- **Writers**: every path that writes the value, as `path:line` or a
  transition ID. Writers outside the repository read `unknown` unless the
  user named them.
- **Copy of** marks a row that restates another row: a counter beside the
  rows it counts, a cached status, a field repeated in a message payload, a
  limit kept per process.
- A value nothing reads on the way to the screen stays in the table, with
  `unobserved` in the readers column.

A status with its own lifecycle is one row here; reconstructing its reachable
states is a different audit (see [Related skills](#related-skills)).

## 2. Transitions: what commutes, what's idempotent, what's order-dependent

List each event that writes a state row: a user action, a request, a
delivered message, a timer, a retry, a recovery job.

| ID | Event | Writes | Guard | Applied twice | Against other transitions |
| -- | ----- | ------ | ----- | ------------- | ------------------------- |

Classify from the code, citing the line that makes it so:

- **Idempotent**: the same event applied twice leaves the same state and
  fires its effects once. Name what makes it so: a guard on the current
  state, a stored key, a unique constraint, a set instead of an increment.
  One event can be idempotent for one row and not for another; say which.
- **Commutes**: two transitions give the same state and the same effects in
  either order.
- **Order-dependent**: the result depends on which lands first. Name what
  decides today: a lock, a transaction, one consumer per key, a sequence
  number, a caller's `await`, or nothing.

Classify pairs only where both orders can happen: transitions that touch the
same rows and whose triggers nothing orders. Write `unknown` instead of
guessing; an unknown cell is a coverage gap, not a finding.

## 3. Dependencies: what must be true before each effect fires

An effect is anything that leaves the program or reaches the screen: a
response, a rendered value, a published message, an email, a charge, a call
to another system, a write another program reads. One row per precondition
(E1a, E1b for two preconditions of one effect). A guard that protects an
effect appears in both tables; the E row cites it.

| ID | Effect | Fired by | Must be true before | Enforcer |
| -- | ------ | -------- | ------------------- | -------- |

The enforcer is one of:

- **here**: a check, constraint, transaction or type in this program, with
  `path:line`. Say what it checks against; a precondition checked against a
  copy is only as good as the copy.
- **elsewhere**: the caller, the database, the broker or another service,
  named, with where the guarantee is stated (code, configuration, a written
  contract) or `unverified`.
- **none found**, with where you searched.

## Then find

Each pass reads the tables. What it turns up is a candidate until it survives
[Check each finding](#check-each-finding).

### Nondeterminism in awaits, retries and event order

For each queue, retry loop, webhook or scheduler the program uses, first
state the delivery semantics the code intends, with the line or
configuration that shows them: at most once or at least once, ordered per
key or unordered, retried without bound or a bounded number of times and
then parked. Where nothing says, do not guess the intent: put the question
under decisions for the owner.

Against that intent, a source of nondeterminism is a candidate only where it
meets a row that cannot absorb it: a repeat (a retry, a redelivery, a double
click) meets a cell that is not idempotent; two events nothing orders meet
an order-dependent pair; an `await` separates an enforcer's check from the
effect it protects.

### Invariants enforced by hope

Hope is a precondition or invariant whose enforcer is **none found**: it
holds only by convention or timing. Report it with the search you made. One
enforced **here** is not a finding. One enforced **elsewhere** is neither
hope nor safe: it goes to the report's contracts, with what the user sees if
that guarantee changes.

### One fact stored in two places, drifting

For each "copy of" row, list every writer of both rows. The copies can drift
when a writer changes one without the other in the same atomic step, when a
retry repeats one write and not the other, or when they have different
lifetimes (a restart, a second replica, a cache expiry). Say which copy is
the source, which one the screen reads, which one guards an effect, and what
the user sees when they disagree.

### Work that never reaches the screen

Start from the `unobserved` rows and from effects nothing reads: a value
written and never read, a result discarded by a later guard, a recovery job
for a state nothing produces, a message with no consumer, an error caught and
dropped, paid work repeated although its first result was kept. Say what the
work costs (time, money, a held lock or slot); cost is its whole impact. "No
reader in this repository" is not "no reader": a possible consumer in another
application is a contract question.

## Check each finding

Candidates from this audit fail in three ways: the code was misread, an
enforcer somewhere else already covered the case, or two candidates shared
one root cause. Before reporting a candidate:

1. **Anchor it.** Name the S, T and E rows it violates.
2. **Write its smallest breaking input** in the format below. Two kinds of
   candidate have none. For unobserved work, give the search for readers and
   the cost. For an invariant nothing breaks today, name the assumption and
   the change that would break it; that finding is latent.
3. **Try to disprove it.** Read the control flow at each cited line
   literally, not by its shape: a `try` around `return send()` catches
   nothing from a promise it did not await. Then look, along the writers and
   guards of the anchored rows, for the check that already rejects a step of
   the input. If it is in this program, drop the candidate and list it under
   ruled out with the `path:line`; if it is elsewhere, move it to contracts.
4. **Merge by root cause.** Candidates whose inputs break at the same row
   and step are one finding with several symptoms. Where fixing one would
   resolve or change another, link them.

### Smallest breaking input

A numbered sequence of events from a named starting state to a wrong result
on the screen:

```text
start: import 7 is processing, 200 lines in two chunks, rows_done = 0
1. chunk 1 delivered: 100 rows inserted, rows_done = 100, committed  consumer.ts:52-58
2. process dies before the ack                                       consumer.ts:58-63
3. chunk 1 redelivered: inserts skipped, rows_done = 200             consumer.ts:52-58, queue.ts:12
4. rows_done reaches 200: status = done, email queued                consumer.ts:66-70
seen: "import complete" while the table holds 100 of 200 lines
expected: completion only after chunk 2
```

- Each step is something a test can drive: a call, a delivered message, a
  second session, a killed process, a clock advance, a stored row. The
  sequence should become the regression test almost unchanged. Prose ("under
  load a message may be processed twice") is not a breaking input.
- Smallest means removing any step makes the failure disappear: one entity,
  and two actors unless the defect needs a third.
- Every step cites the `path:line` that does it. A comment, or a call into
  code you could not read, is not a citation: mark that step `unverified`
  and say what it depends on. Report the count as `4/4 steps cited`.
- If you cannot build an input, say so and name the assumption the code
  rests on. Do not invent one.

## Last, split the refactors into "provably equivalent" and "trust me"

This part takes a diff: a commit range, a branch, a pull request or a patch
the user names. Given none, use the current branch against the default
branch (the remote's, or local `main` or `master` without a remote),
uncommitted changes included, and say that you did. If that diff is empty,
skip the split and list it under not covered with the input it takes; do not
search history for refactors to judge.

Map each change to the model, comparing the rows it touches at the base and
at the head. A change the commit or pull request describes as altering
behaviour is not a refactor; list it apart.

- **Provably equivalent**: no row changes for any input, and the argument
  fits in a line or two: a rename or move, an extraction that keeps the
  calls and their order, a reordering of two steps the transition table
  shows commute. Passing tests are evidence, not the argument.
- **Trust me**: a row changes, or the argument needs an assumption. Typical
  cases: an `await` added, removed or moved, a shifted transaction boundary,
  error handling moved across a call, a changed default, predicate or
  serialisation, anything near retries or acknowledgement.

For each "trust me", give the smallest input that breaks it, in the format
above, with the old result and the new one. If you tried and found none, say
`no breaking input found` and name the assumption the change rests on. A
"trust me" that breaks is a finding: rank it, and report it once, here.

## Report

Write in the user's language. Read the
[worked example](references/worked-example.md) before writing your first
table. Rank findings by what the user sees go wrong, taking `unverified`
steps as true and, where the intent is unknown, the worse reading:

1. **Wrong or lost**: wrong data on the screen, lost work, an effect fired
   twice or never.
2. **Stuck**: something never completes, or only after manual action.
3. **Late or costly**: the result is right, but slower or paid for twice.
4. **Latent**: nothing goes wrong today.

The report has, in this order:

1. A one-sentence summary; the screen, the scope, the revision and the
   intended delivery semantics; and a table of findings numbered in rank
   order: number, title, group, rows, rank, steps cited, fix kind.
2. The model: the three tables.
3. Findings grouped by the model's lifecycles or owners, so that each group
   is one batch to review and ship. Order groups by their highest-ranked
   finding, and findings inside a group so that each follows those it
   depends on. Give rank 3 and 4 one line each unless asked. A rank 1 or 2
   finding has:
   - a plain-language title saying what the user sees;
   - its rows, its breaking input and what you checked while trying to
     disprove it;
   - the fix kind: `bug fix` (restores behaviour the code or its documents
     already state), `behaviour change` (the intent is unstated, or the fix
     changes what users or other systems rely on, such as replacing an
     endless requeue with bounded retries and a dead-letter queue) or
     `external contract` (cannot be fixed inside this program);
   - the smallest fix and what it touches: other writers and readers of its
     rows, test doubles of a changed constructor or port, generated API
     contracts, consumers in other applications, and whether it holds while
     old and new instances run side by side;
   - the findings it depends on or resolves.
4. Decisions for the owner: every `behaviour change` and every unstated
   intent, with the options.
5. Contracts with other systems: every **elsewhere** enforcer and every
   `external contract` finding, with the guarantee needed and from whom (for
   example, an ordering guarantee between two producers).
6. The refactor split, when it ran.
7. Ruled out and not covered: how many candidates you dropped and which
   enforcer settled them, `unknown` cells, lifecycles not modelled, hand-offs
   to a skill that was not available.

The report describes one revision. Before a finding is implemented, by you
or by anyone else, replay its breaking input against the current code: an
earlier fix from the same report may already have closed it.

## Related skills

This skill is complete on its own. Two others go deeper where it stops. Use
one only when it is available in the session. When it is not, do not
reproduce its method from memory: finish this audit, list the hand-off under
not covered and give the install command.

- `find-implicit-state-machines`: when a state row is a lifecycle (a status,
  a phase, a set of flags) whose reachable states need reconstructing, and
  for its checks on queues, workers, leases and durable workflows. Keep this
  report's format and order; add that skill's label and evidence method to
  each finding as extra columns, as it defines them.
  `npx skills add kr4chinin/skills --skill find-implicit-state-machines`
- `convert-state-machines-to-actors`: when a fix has to put several writers
  of one persisted state behind a single owner.
  `npx skills add kr4chinin/skills --skill convert-state-machines-to-actors`
