# `audit`

## Syntax

```text
>>> audit <NodeName>
```

- `<NodeName>` — an existing Node's class name.

## Behavior

1. Resolves `<NodeName>` against the current `agent.topology`, same as
   `do`/`apply`/`build`/`update` (see `do.md`, `../agent.md`).
2. Reads `.class/<NodeName>.md` (the Node's own description, written by
   `create` — see `../Node/class.md`). If it's missing (the Node
   predates this convention, or was never given one), reports that
   explicitly instead of failing.
3. For each of `<NodeName>`'s registered transforms, reads
   `.class/<TransformName>.md` the same way.
4. For the Node, and separately for each transform with a description on
   file, asks `Agent.free(...)` to judge whether the current
   `properties()`/`apply()` body still plausibly implements its stored
   description — not by re-deriving what the properties/behavior *should*
   be from scratch, but by comparing the two and naming any drift.
5. Reports one line per artifact checked (the Node's `properties()`, each
   transform's `apply()`): `- <artifact>: no drift detected` or
   `- <artifact>: <what's drifted, in the agent's own words>`, plus one
   line per artifact whose description file was missing.

## Purpose

`init`'s diagnosis (see `init.md`) is purely structural — a missing file,
a load error, a name that doesn't match. A Node can load and run
perfectly while having drifted completely from what it was originally
asked to do, and `init` would call it clean. `audit` is the semantic
counterpart: a judgment call, not a structural fact, so it's a soft
report the user can act on or ignore — never a hard diagnostic `init`
should block on, and never run implicitly as part of any other command.
