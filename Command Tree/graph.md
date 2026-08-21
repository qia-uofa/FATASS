# `graph`

## Syntax

```text
>>> graph
```

Takes no arguments.

## Behavior

1. Scans `Topology/<agent.cwd>` (see `../agent.md`) for this level's
   Nodes, same as `init`'s scan (see `init.md`, `../Node/nesting.md`'s
   "Enumerating a level"): descending transparently through any plain
   organizational folder, stopping at each Node found — a Node's own
   nested children aren't descended into here, even if it has some;
   `cd` into it and run `graph` again there for a diagram of its
   children.
2. Creates `./out/` if it doesn't exist yet.
3. For each Node, emits a UML element labeled with its path relative to
   `agent.cwd` — not its class name, which is unique but spells out the
   Node's *entire* path from `./Topology` (see `../Node/nesting.md`'s
   "Naming"), unnecessarily verbose for a diagram already scoped to one
   level — as `folder <path>` if it currently has one or more
   nested child Nodes of its own (anywhere inside it, at any depth),
   `node <path>` otherwise, so the diagram visibly distinguishes which
   Nodes have children to `cd` into from ones that don't.
4. For each Node that has a `"build"` transform registered, for every
   Node listed in that transform's `inputs` (each stored as a
   `./Topology`-relative path — see `../Node/transform.md` — reexpressed
   relative to `agent.cwd` for the label, same as step 3), emits a
   directed arrow from the input Node to the Node being built
   (`<InputPath> --> <Path>`) — the same build-dependency edge
   `build -r`/`update -r`/`purge -r` walk (see `build.md` step 3,
   `purge.md` step 3). A Node with no `"build"` transform simply appears
   with no arrows — not an error, since not every Node needs one.
5. Writes the full diagram, wrapped in `@startuml`/`@enduml`, to
   `./out/graph.uml`, overwriting whatever was there before — `graph`
   always regenerates the whole picture from this level's current state,
   never patches the existing file in place.
6. Reports back the path written, and the number of Nodes and arrows
   drawn.

## Purpose

A visual, standalone rendering of the same build-dependency graph
`build -r`, `update -r`, and `purge -r` walk programmatically (see
`build.md`, `update.md`, `purge.md`) — useful for seeing one level's
shape at a glance (which Nodes are leaves, which are terminal outputs,
where a diamond or a cycle sits, which Nodes have children of their own
nested inside them) without having to trace `transform_names()`/`inputs`
through `.class/` by hand.
