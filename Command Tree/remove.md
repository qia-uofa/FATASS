# `remove`

## Syntax

```text
>>> remove <Path>
```

- `<Path>` — what to remove, relative to `agent.cwd` (see `../agent.md`),
  using the same name-shape detection as `create` (see `create.md`):
  - **Node path** — an existing Node.
  - **Transform path** (ends in `.class/<TransformName>.py`) — an
    existing Transform registered on the owning Node.

## Behavior

### Removing a Node (`<Path>` is a Node path)

1. Resolves `<Path>` against `agent.cwd` (see `../Node/nesting.md`,
   `../agent.md`), same as `do`/`apply`/`build`/`update`.
2. Scans every *other* Node anywhere under `./Topology` for one whose
   `inputs` includes this Node — nothing is out of reach here, since any
   Node can reference any other Node by path regardless of where either
   sits in the tree (see `../Node/nesting.md`). A transform registered on
   this Node itself, or on one of its own descendant Nodes (see step 3),
   doesn't count against it — those are being removed right along with
   it, so depending on their own soon-to-be-gone output isn't an external
   dependency. If any transform belonging to a Node outside this Node's
   own subtree still lists it (or one of its descendants) as an input,
   refuses to remove and reports which transform(s) depend on it — the
   fix is to `remove` or `rename` (retarget) those first.
3. Once clear, deletes this Node's own directory entirely, `.class/`
   included — and, since any Node can host children (see
   `../Node/nesting.md`), this also deletes everything nested inside it:
   every descendant Node it hosts (and any plain organizational folder
   leading to one), recursively, with no separate "is it empty first"
   check beyond the dependency scan in step 2 (which already covers this
   Node's whole subtree, not just its own level).

### Removing a Transform (`<Path>` is a Transform path)

1. Resolves the owning Node and confirms `<TransformName>` (`<Path>`'s
   final segment, minus `.py`) is one of its registered transforms (same
   resolution as `do`).
2. Scans every Node anywhere under `./Topology`'s transform modules for
   one whose `apply()` invokes the target transform (via
   `transforms['<TransformName>']` on the owning Node, resolved by path —
   see `../Node/nesting.md`) — a judgment call routed through
   `Agent.free(...)`, not a plain text grep, since the reference might be
   indirect. If any other transform invokes it, refuses to remove and
   reports which transform(s) do — the fix is to update or remove those
   first.
3. Once clear, deletes the transform's own `.py` file and its `.md`
   description file (see `../Node/class.md`), and removes
   `"<TransformName>"` from the owning Node's `transform_names()` list.

## Purpose

The structural inverse of `create`: where `create` adds a `.class/`
definition, `remove` deletes one — but only once nothing else in the
Topology still depends on it, so removal can never silently leave a
dangling `inputs` reference or an unreachable `transforms[...]` lookup
behind. Scoped to structural deletion only; it doesn't touch a Node's
non-`.class/` content beyond deleting the Node wholesale (see `free.md`
for content-level edits, `build.md`/`do.md` for regenerating content).
