# `graph`

## Syntax

```text
>>> graph
```

Takes no arguments.

## Behavior

1. Scans the current `agent.topology` (see `../agent.md`) for every
   existing child Node, same as `init`'s scan (see `init.md`) — the root
   before any `cd`, or whichever Topology `cd` last left `agent.topology`
   pointing at otherwise. Only that one level: a child that's itself a
   topology node is drawn per step 3 below, but its own nested
   `Topology/` isn't descended into — `cd` into it and run `graph` again
   there for a diagram of its children.
2. Creates `./out/` if it doesn't exist yet.
3. For each Node, emits a UML element named after the Node's class name
   — `node <NodeName>` for a plain Node, `folder <NodeName>` for a
   topology node (see `../Node/topology.md`), so the diagram visibly
   distinguishes which Nodes have their own nested Topology to `cd` into.
4. For each Node that has a `"build"` transform registered, for every
   Node listed in that transform's `inputs`, emits a directed arrow from
   the input Node to the Node being built (`<InputNode> --> <NodeName>`)
   — the same build-dependency edge `build -r`/`update -r`/`purge -r`
   walk (see `build.md` step 3, `purge.md` step 3). A Node with no
   `"build"` transform simply appears with no arrows — not an error,
   since not every Node needs one.
5. Writes the full diagram, wrapped in `@startuml`/`@enduml`, to
   `./out/graph.uml`, overwriting whatever was there before — `graph`
   always regenerates the whole picture from the current topology's
   state, never patches the existing file in place.
6. Reports back the path written, and the number of Nodes and arrows
   drawn.

## Purpose

A visual, standalone rendering of the same build-dependency graph
`build -r`, `update -r`, and `purge -r` walk programmatically (see
`build.md`, `update.md`, `purge.md`) — useful for seeing one Topology
level's shape at a glance (which Nodes are leaves, which are terminal
outputs, where a diamond or a cycle sits, which Nodes have their own
nested Topology) without having to trace `transform_names()`/`inputs`
through `.class/` by hand.
