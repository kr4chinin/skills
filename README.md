# skills

Agent skills for Claude Code, Codex, Cursor and other agents supported by the
[`skills`](https://github.com/vercel-labs/skills) CLI.

```sh
npx skills add kr4chinin/skills --skill <name>
```

## find-implicit-state-machines

Finds the implicit state machines in a codebase: states and transitions spread
across flags, status fields, timers and callbacks instead of being declared in
one place. It reports the ones that can get stuck, reach impossible states or
race, each with a state diagram and a fix.

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="assets/find-implicit-state-machines/diagram-dark.svg">
  <img alt="Three flags isLoading, error and data make eight combinations; four are intended. The agent reconstructs the machine and shows that retry leads to an unintended Loading + error state." src="assets/find-implicit-state-machines/diagram-light.svg">
</picture>

```sh
npx skills add kr4chinin/skills --skill find-implicit-state-machines
```

> Find all of the implicit state machines in my app.

The audit is read-only. Each finding is labelled Confirmed, Likely or Smell,
depending on how far it was traced through the code.

## License

MIT
