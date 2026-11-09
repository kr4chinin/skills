# Statechart libraries

Read before using a statechart library the project already has. The
transition stays pure, effects come back as data, and the actor acts on them.
Check every API name against the installed version.

## XState v5

The exported `transition` exists since 5.19. On older versions, or if a name
below is missing, use a plain transition function.

```ts
const machine = setup({ types, guards, actions }).createMachine({ ... });

// First write, no row yet: runs the context factory; entry actions come back
// in firstActions, not executed.
const [first, firstActions] = initialTransition(machine, input);

// Later writes: rebuild from the stored value and the full stored context.
const snapshot = machine.resolveState({ value, context });
if (!snapshot.can(event)) {
  return rejected(); // no transition, or a guard said no
}
const [next, actions] = transition(machine, snapshot, event);
```

- `transition` and `initialTransition` run no action implementations;
  `assign` is already applied. Translate `firstActions` like `actions`, in
  the insert. If the creating event is itself a transition, check
  `first.can(event)` and run `transition(machine, first, event)` before
  inserting, then insert `next` with both action lists.
- Only named actions with serializable `params` become effects, mapped by
  `type` and `params`. `xstate.*` entries (raise, log, emit, send, cancel,
  spawn, stop), anonymous actions or unserializable params put the machine
  outside this recipe. An immediate `xstate.raise` is already applied;
  replaying it doubles the event.
- XState has no rejected result. Decide with `snapshot.can(event)`, which
  runs the guards, so keep them pure. It cannot tell "no transition" from
  "guard failed"; check the state's transitions if callers need both.
  Comparing state values misses a targetless transition that changes only
  context.
- Persist `next.value` and the whole `next.context`; `resolveState` with a
  partial context leaves fields undefined.
- Sound only without history states, invoked or spawned actors, `after`
  delays, delayed `raise`, `sendTo`, `emit` and inline actions:
  `resolveState` rebuilds with no history or children, and a pure transition
  never fires timers, so the process stalls with no error. Model delays as a
  persisted due time plus a scheduler event and external work as an awaiting
  state; otherwise ask the user.
- Use the exported `transition(machine, snapshot, event)`, not
  `machine.transition`. Never call `createActor` or `actor.start`.
