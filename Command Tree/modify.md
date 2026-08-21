# `modify`

## Syntax

```text
>>> modify <Path> "<prompt>"
```

- `<Path>` — the existing Node or Transform to modify, relative to
  `agent.cwd` (see `../agent.md`), using the same name-shape detection as
  `create` (see `create.md`):
  - **Node path** — modifies the Node's properties.
  - **Transform path** (ends in `.class/<TransformName>.py`) — modifies
    the Transform's behavior.
- `<prompt>` — free text describing the change to make: to what the
  Node's properties should be, or to what the Transform should do.
  Folded into the artifact's existing description, not appended as a
  separate note — see Behavior below.

Unlike `create`, `modify` never creates anything: `<Path>` must already
resolve to an existing Node or Transform, same requirement `remove` and
`rename` have (see `remove.md`, `rename.md`).

## Behavior

### Modifying a Node's properties (`<Path>` is a Node path)

1. Resolves `<Path>` against `agent.cwd` (see `../Node/nesting.md`,
   `../agent.md`), same as `do`/`apply`/`build`/`update`. An error if no
   `.class/` exists there yet — `modify` only ever changes an existing
   Node; use `create` (`create.md`) to make a new one.
2. Reads the Node's current description from `.class/<NodeName>.md` (see
   `../Node/class.md`, `<NodeName>` being its own final path segment). If
   it's missing (the Node predates the convention, or was never given
   one), treated as blank rather than failing — same tolerance `audit`
   has (see `audit.md`).
3. Asks `Agent.free(...)` to reconcile `<prompt>` into that description —
   not appended verbatim underneath the old text, but merged the way a
   person editing the doc by hand would: superseding whatever `<prompt>`
   contradicts, keeping whatever it doesn't touch. Produces the Node's
   new, complete description text.
4. Overwrites `.class/<NodeName>.md` with that new text. The file always
   holds the single current description, same as `create` writes it —
   never a running log of every `modify` that's touched it.
5. Regenerates `properties(self)` on the Node's class from the new
   description, the same derivation `create` performs the first time
   (see `create.md`'s Node behavior, step 3) via `Agent.free(...)`,
   replacing the existing method body outright rather than layering the
   new judgment on top of the old one.
6. That `Agent.free(...)` call's access is restricted exactly as
   `create`'s: read-only to every Node's `.class/` directory (for
   consistency with how other Nodes are defined), no read access to any
   Node's content outside `.class/`. Write access confined to this
   Node's own `.class/`.

### Modifying a Transform's behavior (`<Path>` is a Transform path)

1. Resolves the owning Node and confirms `<TransformName>` (`<Path>`'s
   final segment, minus `.py`) is one of its registered transforms — same
   resolution `do` uses.
2. Reads the transform's current description from
   `<OwningNodeDir>/.class/<TransformName>.md` (see `../Node/class.md`).
   Missing is treated as blank, same as above.
3. Reconciles `<prompt>` into it via `Agent.free(...)`, same merge
   discipline as the Node case: superseding what `<prompt>` changes,
   keeping the rest.
4. Overwrites `.class/<TransformName>.md` with the new description.
5. Regenerates `apply(self)` from the new description, the same
   derivation `create` performs the first time (see `create.md`'s
   Transform behavior, step 3), replacing the existing method body.
   `inputs`/`output` are left untouched — `modify` changes what a
   Transform *does* with its bindings, never what they point at;
   retargeting them is a `remove` plus `create` (or `rename`), not a
   `modify`.
6. Access restricted exactly as `create`'s Transform behavior: read-only
   to every Node's `.class/` directory, no read access to content outside
   `.class/`, write access confined to the owning Node's own `.class/` —
   the transform module being edited.

## Purpose

`create` (`create.md`) derives a Node's `properties()` or a Transform's
`apply()` from a description once, at creation time. Descriptions drift
from that generated logic over time regardless — that's what `audit`
(`audit.md`) surfaces. `modify` is how the drift gets *fixed*, or the
description deliberately *changed*, without going through `remove` +
`create` and losing the artifact's identity (its name, its place in the
tree, its `inputs`/`output` bindings) along the way: it keeps the
description file and the generated method in sync in one step, the same
way `create` does the first time, so an `audit` run immediately
afterward should report no drift.

Distinct from `free` (`free.md`): `free` never touches `.class/` and
edits a Node's actual content directly, unscoped to any generated
method; `modify` never touches a Node's content and only ever
regenerates the `.class/` logic — `properties()` or `apply()` — that a
description drives. Distinct from `rename` (`rename.md`): `rename`
changes an artifact's name everywhere it's referenced, never its
behavior; `modify` changes behavior, never a name.
