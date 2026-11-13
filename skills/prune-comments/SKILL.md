---
name: prune-comments
description: Audit code comments and plan which to delete, keep, rewrite or replace with a refactor. Use when asked to prune or clean up comments, or to find workaround and hack comments worth fixing at the root.
---

# Prune comments

Agents copy what a codebase already does, so every comment is a _precedent_.
A comment that narrates the code teaches narration; one that excuses a
temporary workaround teaches that workarounds are fine here. Prune until the
comments that remain are precedents worth copying: each states a _why_ the
code cannot express.

The first deliverable is a plan. Edit code only after the user approves it.

## 1. Scope

Use the scope the user gave; with several apps and no hint, ask which. Cover
files tracked by git, and report the revision you read. Leave out generated,
vendored and third-party code, lockfiles and snapshots, and say what you left
out.

Find comments with a syntax-aware tool (`ast-grep` with `kind: comment`, or
the language's parser) when one is available: text search also matches `//`
and `#` inside strings and URLs. Doc comments and docstrings count as
comments.

## 2. Classify every comment

Give each comment one verdict. Judge it against the code beside it, the
history of those lines (`git log -L`, `git blame`) and any linked issue: the
code changes since the comment was written show whether it is stale.

**Keep**: the comment carries information the code cannot.

- An external quirk: browser, OS, runtime, library or API behaviour, ideally
  with a link or version.
- A constraint or invariant that is not obvious, with what breaks without it:
  ordering, concurrency, security, a performance budget, a protocol rule.
- Why the obvious alternative was rejected.
- Public API docs read outside the file, and docstrings that hold doctests.
- Comments the tooling reads: lint and type suppressions, formatter pragmas,
  build tags, `// @ts-check`, `/// <reference>`, `# type:`, shebangs, license
  headers, generated-file markers. A suppression without a reason also gets a
  Rewrite that adds one.

**Delete**: removing it loses nothing.

- It restates the code or narrates the next line.
- Commented-out code; git keeps it.
- Change history, authorship, dates, "added for X".
- Section banners in a file short enough to read whole.
- It describes code that no longer exists or now behaves differently.
- A workaround whose cause is gone: the upstream bug is fixed in the
  installed version, or the code it guarded was removed. The workaround code
  goes with the comment; confirm the fix against the installed version first.
- A TODO or FIXME with no owner, ticket or still-true reason. If it names a
  real defect, list it in the plan as a ticket to file.

**Refactor**: the comment compensates for code that should say it itself. The
fix is a code change, and the comment goes in the same change.

- A comment explaining a variable or function → a better name.
- A comment labelling a block → an extracted, named function.
- A comment explaining a literal → a named constant.
- "May be null", "only one of these is set", "call init first" → types, a
  tagged union, a constructor.
- A hack or temporary workaround for this repository's own code → fix the
  root cause.

A workaround for an external cause the repository cannot change is Keep; make
sure its comment names the cause.

**Rewrite**: the comment should exist but is vague, wrong, or says what
instead of why. Give the replacement text.

Done when every comment in scope has a verdict or is listed as unreviewed
with the reason. On a large codebase, classify the mechanical classes
(commented-out code, banners, restated code) by pattern with counts and
sample them, and read individually every comment that needs judgement:
workarounds, TODOs, anything near a Keep signal.

## 3. Plan

Write in the user's language:

1. Summary: counts per verdict, per area.
2. Refactor, workarounds and hacks first, since they are the precedents agents
   copy most: `path:line`, the comment, what it compensates for, the proposed
   change, effort (S/M/L), risk and the check that will prove it safe. Group
   items that share one fix.
3. Delete, grouped by pattern: count, two or three examples, and whether the
   removal is mechanical. List workaround code that goes with its comment
   separately, as a behaviour change.
4. Rewrite: `path:line`, current text, replacement.
5. Keep: the patterns kept and why, with an example each.
6. Tickets to file, and what was not covered.

Then ask which parts to apply.

## 4. Apply, when asked

Order: deletions, then rewrites, then refactors.

- Deletions go one pattern per commit and touch comments only. Removing a
  workaround is its own change, with a test for the case it handled.
- Each refactor is its own change and deletes its comment.
- After each batch, run the project's formatter, linter, type check and
  tests. A comment-only change that alters any result removed something the
  tooling reads (a suppression, a docstring that was a function's only body,
  a doctest); restore it.

Done when every check is as green as before the batch; report what the checks
do not cover.

## 5. Keep it pruned

Offer a comment rule for the repository's `AGENTS.md` or `CLAUDE.md`, built
from the Keep patterns and stated positively, for example:

> Comments state why the code cannot say it: an external quirk with a link, a
> constraint and what breaks without it, a rejected alternative. Names, types
> and functions carry everything else. Temporary work lives in the issue
> tracker.

Add it only when the user agrees.
