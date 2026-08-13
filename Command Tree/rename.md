# `rename`

## Syntax

```text
>>> rename <Name> <NewName>
```

- `<Name>` — the existing Node or Transform to rename, using the same
  name-shape detection as `create` (see `create.md`):
  - `<NodeName>` (no dot) — renames a Node.
  - `<NodeName>.<TransformName>` (one dot) — renames a Transform
    registered on `<NodeName>`.
- `<NewName>` — the replacement name, in the same shape as `<Name>`: a
  bare name when renaming a Node, `<NodeName>.<NewTransformName>` when
  renaming a Transform. The `<NodeName>` prefix must match `<Name>`'s —
  `rename` changes a name in place, it can't move a Transform to a
  different Node (that's a `remove` on the old Node plus a `create` on
  the new one). Must be a valid Python class name; when renaming a Node,
  also may not be `topology` (case-insensitively), same as `create` —
  that name stays reserved for the root Topology no matter which command
  is asked to produce it (see `../Node/topology.md`).

## Behavior

### Renaming a Node (`<Name>` has no dot)

1. Resolves `<NodeName>` against the current `agent.topology`, same as
   `do`/`apply`/`build`/`update` (see `do.md`, `../agent.md`).
2. Scans every Node in the current `agent.topology`'s `.class/` for
   occurrences of `<NodeName>` that need updating: the class file's own
   class name, every transform's `inputs`/`output` (any
   `Topology().load_node_module("<NodeName>")` call, `<NodeName>`'s own
   transforms included), and any `Agent.free(...)`-authored prose that
   names `<NodeName>` — a judgment call, not a plain text find/replace,
   since references can be indirect. If `<NodeName>` is a topology node,
   nothing inside its own nested `Topology/` needs updating — names
   there are scoped to that nested Topology, unaffected by what its
   host Node is called.
3. Renames the directory `<NodeName>` to `<NewName>` (within the current
   `agent.topology`), renames the class file `<NodeName>.py` to
   `<NewName>.py` and its description file `<NodeName>.md` to
   `<NewName>.md` (see `../Node/class.md`), and updates the class name
   inside the class file to `<NewName>`.
4. Rewrites every occurrence found in step 2 to `<NewName>`.

### Renaming a Transform (`<Name>` is `<NodeName>.<TransformName>`)

1. Resolves `<NodeName>` and confirms `<TransformName>` is one of its
   registered transforms (same resolution as `do`).
2. Scans every Node in the current `agent.topology`'s transform modules
   for occurrences of `<TransformName>` that need updating: `<NodeName>`'s
   `transform_names()` list, and any other transform that invokes
   `<NodeName>.transforms['<TransformName>']` (same judgment-call scan
   `remove` uses to check for dependents — see `remove.md`).
3. Renames `<NodeName>/.class/<TransformName>.py` to
   `<NodeName>/.class/<NewTransformName>.py` (updating the class name
   inside it) and `<NodeName>/.class/<TransformName>.md` to
   `<NodeName>/.class/<NewTransformName>.md`.
4. Rewrites every occurrence found in step 2 to `<NewTransformName>`.

## Purpose

Keeps a Node or Transform's name consistent everywhere it's referenced
— its own `.class/` files, every other Node's `inputs`/`output`
bindings, `transform_names()` entries, and description files — so a
rename never leaves a stale name behind the way manually editing
`./Topology` would. Unlike `remove`, `rename` has no "nothing else may
depend on it" precondition: every dependent gets rewritten instead of
being required to already be gone.
