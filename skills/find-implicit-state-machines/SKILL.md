---
name: find-implicit-state-machines
description: Find implicit state machines in a codebase (state spread across flags, status fields, timers and callbacks), audit for defects, or catalog identified machines when explicitly asked for an inventory. Use when asked to find implicit state machines or audit for impossible states, stuck loading, missing resets or async races.
---

# Find implicit state machines

An implicit state machine has states and transitions that are not declared in
one place but spread across variables, handlers, callbacks and timers. Find
and reconstruct them to audit for defects or build an explicitly requested
inventory. Finding a machine does not mean finding a bug.

Stay read-only: don't edit code or create tickets unless asked.

## 1. Scope

Use the scope the user gave. If the repository holds several apps and there is
no hint, ask which one. Report what you included and skipped.

Default to a defect audit: report the broken or fragile machines. When the user
explicitly requests an inventory (for example, "catalog all machines, including
ones with no defect"), include a compact catalog with locations and review outcomes,
including "no defect found" for fully traced machines without a finding. Retain
these in the catalog even when dropped from defect findings. Continue
investigating suspected defects and use the evidence labels below for findings.

In either mode, distinguish discovery scope (where you searched), fully traced
paths, and unreviewed candidates. State coverage gaps; search hits and file
counts do not establish exhaustive coverage.

## 2. Find candidates

Prioritize units (module, component, class, service) where two or more of these
signals meet; a concrete reachable failure can warrant tracing a candidate with
only one signal:

- several booleans describing one process;
- a status, phase or step field assigned from many places, with no transition
  table;
- mutable references or instance fields used as state;
- flags that are set once and never reset;
- queues, workers, leases or persisted workflows; for these backend candidates,
  read [distributed lifecycle checks](references/distributed-lifecycles.md) to
  trace ownership, persistence and recovery across process boundaries;
- hand-written polling, retry, debounce or autosave loops;
- cancel, reset or close that clears some fields but not all;
- async results applied without checking they are still current;
- state reset by comparing against a stored previous value;
- mutually exclusive UI elements, each with its own flag;
- the same flow copied into several places.

Signal count or complexity alone is not a defect. An existing enum, reducer or
statechart does not rule out implicit coordination across components. A lone
loading flag or ordinary framework-managed state without evidence of a failure
or fragile coordination is still insufficient.

On a large codebase, delegate read-only searches only when authorized,
compatible with the user's constraints, and supported by available tools;
otherwise do the audit yourself. Split bounded, disjoint scopes by domain or
lifecycle. Assign shared code an explicit owner and identify cross-scope
handoffs so complete paths can be traced. Give each subagent its scope, relevant
candidate heuristics and verification requirements, including applicable
references; no recursive delegation unless the user explicitly requests it.

For suspected defects, ask for compact leads: the intended invariant, state
variables, complete event sequence with `path:line` for each step, protections
checked, any unknown step, smallest fix, and scope limits. Return ruled-out
candidates separately with the protections that resolved them, without inventing
a failure or fix. The parent validates and deduplicates leads before selecting
final findings; generate diagrams only after that selection.

## 3. Reconstruct and verify

For each candidate, state the intended invariant and trace a concrete, reachable
event sequence that would violate it. Read the whole code path and write down
the state variables, intended states, reachable combinations, events (user
actions, responses, errors, timers, teardown) and transitions. Check for:

- an impossible combination that is reachable;
- a state with no way out;
- an incomplete reset;
- a race: a stale result overwrites newer state, or one action runs twice;
- work lost on teardown, or timers left running;
- a loop with no terminal or error state;
- copies that have drifted apart.

Actively try to disprove the sequence: check callers and their guards, remounts
(component keys, route changes), disabled or hidden UI, worker/consumer
registration, locks, transactions and concurrency tokens. Trace recovery paths
and check whether degradation or manual recovery is intentionally accepted
before treating it as a defect. Verify framework assumptions against installed
versions and relevant source or official documentation, following the
repository's documentation policy.

If uncertainty remains, choose the smallest meaningful execution within the
audit's authorized scope, preferring local isolated checks. This skill grants
no access to production or other shared systems. If a check is unavailable or
unauthorized, keep the finding Likely and name the unresolved step. A complete
static trace needs no runtime reproduction. Distinguish tests read from tests
run, and state what an executed check actually exercises: a source-method call
or fake alone is not broker, database or SDK proof.

Label each finding by the code trace; state any doubt about user impact in the
text:

- **Confirmed**: you traced every step of the failing sequence in the code.
  A complete static trace is sufficient; it does not imply runtime reproduction
  or require integration or production execution.
- **Likely**: one step depends on runtime behaviour you could not confirm. Name
  that step.
- **Smell**: works today, but only by convention, ordering or a guarantee
  elsewhere. Name it.

Report evidence method separately: `static trace`, `isolated execution`,
`integration execution` or `production evidence` (list all used). Report
impact/severity separately from confidence and evidence method.

Isolated execution of extracted methods or fakes does not validate SDK,
transport or deployment assumptions. Keep a finding Likely when its failing
sequence depends on an unverified step, and name that step. Include
runtime/version, command and artifact details only for checks actually run.

Drop candidates that turn out to be fine, and report how many you dropped. Don't claim anything was run or
reproduced unless you did it.

## 4. Report

When asked to publish findings, preserve their confidence, evidence method,
impact/severity and caveats alongside the matching native Mermaid and source
links pinned to the audited revision. Transfer them only into tasks or documents
the user authorized creating or updating. Verify saved source
separately from rendering and visibility at the destination. If renderer or
destination verification is unavailable, state that limit; do not substitute
PNG unless requested.

Write in the user's language. Order findings Confirmed, then Likely, then
Smell; within each group, user-facing flows first.

1. A one-sentence summary and a table: number, `path:line`, machine, problem,
   label, evidence method, impact/severity.
2. For each Confirmed or Likely finding:
   - a plain-language title stating what happens, then the triggering
     condition, consequence (including what the user sees) and expected
     behaviour; avoid jargon-only titles;
   - compact source evidence: the state variables and failing event sequence,
     with `path:line` for each step and any unconfirmed runtime step named;
   - a Mermaid `stateDiagram-v2` of the actual behaviour with the bad state
     marked by a `classDef`; for a timing race, a `sequenceDiagram` instead;
   - the smallest fix, and a regression test that replays the sequence.
3. Group findings that share a cause and fix, regardless of copy count. Explain
   one representative in full and list every affected location with any
   meaningful differences; do not hide distinct failure sequences in a group.
4. Smells, briefly naming the contract they rely on. Expand them or create
   follow-up tasks only when requested.
5. What was not covered.

### Diagram readability and publishing

Use native Mermaid blocks, keep diagrams under about 12 states, and match the
state names in the text. For example, a stale response that leaves loading set:

```mermaid
stateDiagram-v2
    [*] --> Idle
    Idle --> Loading: submit
    Loading --> Done: response
    Loading --> Failed: error
    Failed --> Loading: retry
    Loading --> Stuck: stale response ignored, loading never cleared
    class Stuck bad
    classDef bad fill:#fee2e2,stroke:#b91c1c,color:#111827
```

Check final rendered diagrams in light and dark themes, especially highlighted
states, notes and sequence regions. Pair every custom fill with an explicit
contrasting text colour. Use translucent sequence highlights so the theme's text
stays readable on its background.

Publish only when explicitly requested. Preserve editable Mermaid; do not
replace it with PNG unless requested. Verify that diagrams render in the
destination with readable labels and contrast in both themes before claiming
publication complete. Saved source alone does not prove visibility. If a
rendering check is unavailable, state which check remains unverified.

## Fixes

Recommend the lightest change that removes the defect:

- one tagged union or enum instead of parallel flags;
- one transition function when transitions are scattered;
- a statechart library only for hierarchy, parallel regions or timers;
- a request id, generation counter or cancellation for races;
- one owner per loop, with an attempt limit and a terminal error state,
  stopped on teardown;
- re-creating the unit instead of resetting its fields when its identity
  changes;
- one shared module for copied flows.

For distributed persistence defects, consider:

- conditional writes/CAS guarded by current status, version or owner, checking
  affected rows so stale completions cannot overwrite newer state;
- transactions for related persistence changes that must commit together;
- persisted retries or wakeups and reconciliation for interrupted work,
  preserving idempotency and using bounded batches that cover the full eligible
  recovery scope without starving later records.
