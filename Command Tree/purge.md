# `purge`

## Syntax

```text
>>> purge [-r] <NodeName>
```

- `<NodeName>` — an existing Node's class name.
- `-r` — also purge, recursively, every Node reachable by walking
  `transforms['build'].inputs` starting from `<NodeName>` (`<NodeName>`'s
  own build-inputs, their build-inputs, and so on). Without `-r`, only
  `<NodeName>` itself is purged.

## Behavior

1. Resolves `<NodeName>` against the current `agent.topology`, same as
   `do`/`apply`/`build`/`update` (see `do.md`, `../agent.md`).
2. Empties `<NodeName>`'s own content — everything under it except
   `.class/` — the same unconditional purge `build` performs (via this
   very command) as its own step before calling `apply()` (see
   `build.md`), but `purge` stops there: it never calls `apply()`
   afterward, it only deletes.
3. If `-r` was given, looks up `transforms['build']` on `<NodeName>`'s
   instance. If it has one, resolves each Node named in that transform's
   `inputs` and recursively repeats steps 2–3 for each of them — so
   `purge -r` walks the whole build-dependency graph (see `graph.md` to
   visualize it), purging every ancestor's content along the way, not
   just `<NodeName>`'s own immediate inputs. If `<NodeName>` has no
   `"build"` transform, recursion has nothing to walk and stops there —
   that's not an error, since not every Node needs one.
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
