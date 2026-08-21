# `purge`

## Syntax

```text
>>> purge [-r] <NodePath>
```

- `<NodePath>` — an existing Node, relative to `agent.cwd` (see
  `../agent.md`).
- `-r` — also purge, recursively, every Node reachable by walking
  `transforms['build'].inputs` starting from this Node (its own
  build-inputs, their build-inputs, and so on). Without `-r`, only this
  Node itself is purged.

## Behavior

1. Resolves `<NodePath>` against `agent.cwd`, same as
   `do`/`apply`/`build`/`update` (see `do.md`, `../Node/nesting.md`,
   `../agent.md`).
2. Empties this Node's own content — everything under it except
   `.class/` *and* except any nested child Node (any subfolder that is,
   or leads down to, a Node — see `../Node/nesting.md`), which are left
   untouched exactly like `.class/` is. This is what keeps purging a
   Node from ever silently deleting a Node nested inside it — the same
   unconditional purge `build` performs (via this very command) as its
   own step before calling `apply()` (see `build.md`), but `purge` stops
   there: it never calls `apply()` afterward, it only deletes.
3. If `-r` was given, looks up `transforms['build']` on this Node's
   instance. If it has one, resolves each Node named in that transform's
   `inputs` (each stored as a `./Topology`-relative path — see
   `../Node/transform.md`) and recursively repeats steps 2–3 for each of
   them — so `purge -r` walks the whole build-dependency graph (see
   `graph.md` to visualize it), purging every ancestor's content along
   the way, not just this Node's own immediate inputs. If this Node has
   no `"build"` transform, recursion has nothing to walk and stops there
   — that's not an error, since not every Node needs one.
4. Keeps track of every Node already purged during this call, so a
   shared or circular build-dependency (two Nodes whose `"build"`
   transforms both list the other, or a diamond where multiple Nodes
   share a common input) gets purged once, not repeatedly or forever.
5. Never touches `.class/` at any level — `purge` is a content-only
   operation, never the structural deletion `remove` does (see
   `remove.md`).
6. Reports back the full list of Nodes purged.

## Purpose

`build` always purges a Node's content before regenerating it (see
`build.md`), but that purge is only ever a means to an end there — this
command is useful on its own when you want a Node's (and, with `-r`, its
whole build-dependency chain's) stale content gone without triggering a
rebuild, e.g. before reworking a transform's `apply()` or freeing up
space. Scoped to content only: it never touches `.class/`, so a purged
Node stays exactly as buildable afterward as it was before — running
`build`/`apply` again just regenerates what was purged.
