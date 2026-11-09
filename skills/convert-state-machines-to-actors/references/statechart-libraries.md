# Statechart libraries

Read this before writing the machine with a statechart library the project
already uses. The rule for any library: the transition is pure, effects come
back as data, and the actor decides what to do with them. Verify every API
name against the installed version and its documentation before using it.

## XState v5

The exported `transition` function exists since 5.19. On older versions, or
if a name below is missing, stop and use a plain transition function.

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

- `transition` and `initialTransition` execute no action implementations.
  `assign` is already applied to the returned context. Translate
  `firstActions` like `actions`, in the same write that inserts the first
  state.
- Only named actions with serializable `params` become effects (an outbox row
  or an awaiting state), mapped by `type` and `params`. An `xstate.*` entry
  (`xstate.raise`, `xstate.log`, `xstate.emit`, send, cancel, spawn, stop), an
  anonymous `actions` entry or unserializable params mean the machine is
  outside this recipe. An immediate `xstate.raise` is already reflected in
  the returned snapshot; replaying it applies the event twice.
- XState has no rejected result: an unhandled event returns the same
  snapshot. Decide rejection with `snapshot.can(event)`, which runs the
  guards, so keep guards pure. `can` does not tell "no transition" from "guard
  failed"; check the state's transitions yourself if callers need both.
  Comparing state values misreads a targetless transition that changes only
  context.
- Persist `next.value` and the whole `next.context`. `resolveState` with a
  partial context leaves the missing fields undefined; defaults and context
  factories do not run.
- This recipe is sound only for machines without history states, invoked or
  spawned actors, `after` delays, delayed `raise`, `sendTo`, `emit` and
  inline actions. `resolveState` rebuilds with empty history and no children,
  and a pure transition never fires timers or delivers invoked results: the
  process stops in a non-final state with no error. Model delays as a
  persisted due time plus a scheduler event, and external work with an
  awaiting state. A machine that truly needs these features is a departure
  from the contract: ask the user.
- Use the exported `transition(machine, snapshot, event)`, not the method
  `machine.transition`, which takes an internal third argument. Never call
  `createActor` or `actor.start` for this.
