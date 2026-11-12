# Report layout

Read before writing the report. The layout comes first, then a worked
example.

## Rank

Rank a finding by what the user sees when its breaking input runs. Count
`unverified` steps as true; the steps-cited count carries the doubt.
Unstated intent does not lower the rank: it makes the fix a `behaviour
change`.

1. **Wrong or lost**: wrong data on the screen, lost work, an effect fired
   twice or never.
2. **Stuck**: something never completes, or only after manual action.
3. **Late or costly**: the result is right, but slower or paid for twice.
4. **Latent**: nothing goes wrong today.

## Sections, in order

1. A one-sentence summary; the screen, the scope, the revision and the
   intended delivery semantics.
2. The model: the three tables, once per lifecycle if you built several.
3. Findings. First an index numbered in rank order: number, title, group,
   rows, rank, steps cited, fix kind. Then the findings, grouped by the
   model's lifecycles or owners so that each group is one batch to review
   and ship. Order groups by their highest-ranked finding, and findings
   inside a group so that each follows those it depends on. Rank 3 and 4
   get one line each unless asked. A rank 1 or 2 finding has:
   - a plain-language title saying what the user sees;
   - its rows, its breaking input and what you checked while trying to
     disprove it;
   - the fix kind: `bug fix` (restores behaviour the code or its documents
     already state), `behaviour change` (the intent is unstated, or the fix
     changes what users or other systems rely on, such as how retries end)
     or `external contract` (cannot be fixed inside this program);
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

## Worked example

A small file-import service. The domain and the findings are invented: take
the form, not the content.

**Summary:** a redelivered chunk counts its lines twice, so an import can be
announced as complete with half of its lines missing; a refactor on this
branch also lets a finished import flip to cancelled.

**Screen:** the import page (status, progress bar, row table) and the
completion email. **Scope:** `src/imports/` at `3f2a91c`, the head of
`main..HEAD`. **Intended delivery:** at least once, unordered, manual ack,
requeued without a bound (`queue.ts:12-15`).

### Model

| ID | Value | Lives in | Writers | Readers on the way to the screen | Source, or copy of |
| -- | ----- | -------- | ------- | -------------------------------- | ------------------ |
| S1 | import status | `imports.status` | T1, T3, T4 | import page | source |
| S2 | lines done | `imports.rows_done` | T2 | progress bar; guard of T3 | copy of S3 |
| S3 | imported rows | `import_rows`, unique on (import, line) | T2 | row table | source |
| S4 | per-line warnings | `import_rows.warnings` | T2 | `unobserved` | source |

| ID | Event | Writes | Guard | Applied twice | Against other transitions |
| -- | ----- | ------ | ----- | ------------- | ------------------------- |
| T1 | upload accepted | S1 = processing | upload id is the primary key, `api.ts:22` | idempotent | none |
| T2 | chunk delivered | S3 insert, S4, S2 += n | locks the import row, skips unless processing, `consumer.ts:50` | S3 idempotent (unique key, `consumer.ts:52`); S2 not (`consumer.ts:58`) | commutes with T2 of other chunks (`rows_done = rows_done + n` in SQL, `consumer.ts:58`); order-dependent with T4, decided by the row lock and the guard: a chunk after the cancel is skipped |
| T3 | S2 reaches the line count | S1 = done | `WHERE status = 'processing'`, `consumer.ts:66` | idempotent | order-dependent with T4, and nothing covers T4's read: T3 after T4's write changes no row; T4's read before T3 and its write after is F2 |
| T4 | user cancels | S1 = cancelled | reads the status and answers 409 unless it is processing, `api.ts:38`; the write has no condition, `api.ts:41` | idempotent | see T3 |

| ID | Effect | Fired by | Must be true before | Enforcer |
| -- | ------ | -------- | ------------------- | -------- |
| E1a | completion email | T3 | queued once per import | here: outbox row written only when T3's update changed a row, `consumer.ts:70` |
| E1b | completion email | T3 | every line is in S3 | here, but checked against S2, a copy: `consumer.ts:66` |
| E2 | chunk messages | T1 | the file is readable by the consumer | elsewhere: object storage read-after-write, `unverified` |

### Findings

| # | Title | Group | Rows | Rank | Steps cited | Fix kind |
| - | ----- | ----- | ---- | ---- | ----------- | -------- |
| F1 | "Import complete" is sent while lines are still missing | chunk consumer | S2, S3, T2, T3, E1b | 1 | 4/4 | bug fix |
| F2 | A finished import flips to cancelled after its "complete" email | cancel | S1, T3, T4 | 1 | 3/3 | bug fix, see the refactor split |
| F3 | Per-line warnings are computed and stored, and nothing shows them | chunk consumer | S4, T2 | 3 | no reader in `src/` | behaviour change |

#### Chunk consumer

**F1. "Import complete" is sent while lines are still missing** (rank 1)

Rows: S2 is a copy of S3; T2 is idempotent for S3 and not for S2; T3 and
E1b check the copy.

```text
start: import 7 is processing, 200 lines in two chunks, rows_done = 0
1. chunk 1 delivered: 100 rows inserted, rows_done = 100, committed  consumer.ts:52-58
2. process dies before the ack                                       consumer.ts:58-63
3. chunk 1 redelivered: inserts skipped, rows_done = 200             consumer.ts:52-58, queue.ts:12
4. rows_done reaches 200: status = done, email queued                consumer.ts:66-70
seen: "import complete" while the table holds 100 of 200 lines
expected: completion only after chunk 2
```

Checked: no delivery id is stored; the unique key covers S3 only; nothing
recomputes S2 from S3.

Fix (bug fix): add to `rows_done` the number of rows the insert created.
Touches: the consumer's tests that expect `+ n`; S2 has no other writer. In
a mixed rollout old instances still add `n`, so it holds only once they are
gone. Depends on nothing; F3 changes the same handler, so ship F1 first.

**F3** (rank 3): one validation pass per line; no reader of S4 in `src/`.

#### Cancel

**F2. A finished import flips to cancelled after its "complete" email**
(rank 1): introduced on this branch; input and fix under the refactor split.

### Decisions for the owner

- F3: show the warnings, or stop computing them?

### Contracts with other systems

- E2 relies on read-after-write in object storage. If that weakens, the
  consumer cannot read the file and the import stays at processing.

### Refactor split (`main..HEAD`)

| Change | Class | Argument or breaking input |
| ------ | ----- | -------------------------- |
| `parseLine` extracted into its own module | provably equivalent | a move: same call, same arguments, no row touched |
| cancel handler reuses `setStatus` instead of its own conditional update | trust me (F2) | T4's guard is no longer atomic with its write; input below |

```text
start: import 7 is processing, its last chunk is in flight
1. cancel handler reads status = processing            api.ts:38
2. last chunk delivered: status = done, email queued   consumer.ts:50-70
3. cancel handler writes status = cancelled            api.ts:41
seen (new): "import complete" email, and the page shows cancelled
seen (old): cancel answers 409 and the import stays done
```

Checked: `setStatus` writes by id alone (`status.ts:9`), and the handler
takes no lock between its read and its write.

Fix (bug fix): keep the condition in the write (`WHERE status =
'processing'`) and answer 409 when it changes no row. Touches: the other
callers of `setStatus`. Old instances still have the conditional update, so
it holds in a mixed rollout.

### Ruled out and not covered

- Ruled out, 2. "T3 applied twice queues two emails": the outbox row is
  written only when the guarded update changed a row (`consumer.ts:70`). "A
  chunk lands after cancel": T2 locks the import row and skips unless it is
  processing (`consumer.ts:50`).
- Not covered: the outbox relay (not modelled); readers of
  `import_rows.warnings` outside this repository (`unknown`); the reachable
  states of the status lifecycle, a hand-off to
  `find-implicit-state-machines`, which is not installed
  (`npx skills add kr4chinin/skills --skill find-implicit-state-machines`).
