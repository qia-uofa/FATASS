# `remove`

## Syntax

```text
>>> remove <Name>
```

- `<Name>` — what to remove, using the same name-shape detection as
  `create` (see `create.md`):
  - `<NodeName>` (no dot) — an existing Node.
  - `<NodeName>.<TransformName>` (one dot) — an existing Transform
    registered on `<NodeName>`.

## Behavior

### Removing a Node (`<Name>` has no dot)

1. Resolves `<NodeName>` against the current `agent.topology`, same as
   `do`/`apply`/`build`/`update` (see `do.md`, `../agent.md`).
2. Scans every *other* Node in the current `agent.topology` for one
   whose `inputs` includes `<NodeName>` — a transform on a Node outside
   `agent.topology` can't reference `<NodeName>` in the first place, so
   there's nothing further out to check. A transform registered on
   `<NodeName>` itself doesn't count against it — that transform is
   being removed right along with the Node it belongs to, so depending
   on its own soon-to-be-gone output isn't an external dependency. If
   any transform belonging to a different Node still lists `<NodeName>`
   as an input, refuses to remove and reports which transform(s) depend
   on it — the fix is to `remove` or `rename` (retarget) those first.
3. Once clear, deletes `<NodeName>` entirely, `.class/` included. If
   `<NodeName>` is a topology node (see `../Node/topology.md`), this
   also deletes everything nested under its own `Topology/` — every
   descendant Node it hosts, recursively, with no separate "is it empty
   first" check beyond the sibling-dependency scan in step 2 (which
   already only concerns `<NodeName>`'s own level, not its descendants).

### Removing a Transform (`<Name>` is `<NodeName>.<TransformName>`)

1. Resolves `<NodeName>` and confirms `<TransformName>` is one of its
   registered transforms (same resolution as `do`).
2. Scans every Node in the current `agent.topology`'s transform modules
   for one whose `apply()` invokes
   `<NodeName>.transforms['<TransformName>']` (a judgment call routed
   through `Agent.free(...)`, not a plain text grep, since the reference
   might be indirect). If any other transform invokes it, refuses to
   remove and reports which transform(s) do — the fix is to update or
   remove those first.
3. Once clear, deletes `<NodeName>/.class/<TransformName>.py` and its
   `<NodeName>/.class/<TransformName>.md` description file (see
   `../Node/class.md`), and removes `"<TransformName>"` from
   `<NodeName>`'s `transform_names()` list.

## Purpose

The structural inverse of `create`: where `create` adds a `.class/`
definition, `remove` deletes one — but only once nothing else in the
Topology still depends on it, so removal can never silently leave a
dangling `inputs` reference or an unreachable `transforms[...]` lookup
behind. Scoped to structural deletion only; it doesn't touch a Node's
non-`.class/` content beyond deleting the Node wholesale (see `free.md`
for content-level edits, `build.md`/`do.md` for regenerating content).
