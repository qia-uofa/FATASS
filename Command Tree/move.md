# `move`

## Syntax

```text
>>> move <NodePath> <NewParentPath>
```

- `<NodePath>` — the existing Node to move, relative to `agent.cwd` (see
  `../agent.md`). Unlike `rename`, `move` never operates on a Transform
  path — a Transform's location is fixed to its owning Node's `.class/`;
  relocating one to a different Node is a `remove` (`remove.md`) plus
  `create` (`create.md`) on the destination instead.
- `<NewParentPath>` — where `<NodePath>` should be relocated to, relative
  to `agent.cwd`: the new parent directory it becomes nested under,
  keeping its own final segment (folder name) unchanged — changing that
  is `rename`'s job (`rename.md`), not `move`'s. May be `.` (`agent.cwd`
  itself), an existing plain organizational folder, or an existing
  Node's own directory (nesting a Node inside another Node, same as
  `create` allows — see `../Node/nesting.md`). Any missing intermediate
  segment is created as a plain organizational folder along the way,
  same as `create`.

`rename` changes a Node's own final segment in place and explicitly
never relocates it to a different parent (see `rename.md`); `move` is
the complement — it only ever relocates, never touches the final
segment. Doing both at once is a `move` followed by a `rename`.

## Behavior

1. Resolves `<NodePath>` and `<NewParentPath>` against `agent.cwd` (see
   `../Node/nesting.md`, `../agent.md`), same as `do`/`apply`/`build`/
   `update`.
2. Rejects the move outright if `<NewParentPath>` is `<NodePath>` itself,
   or sits inside `<NodePath>`'s own directory at any depth — relocating
   a Node underneath itself is a cycle, not a valid nesting. Also rejects
   any segment of `<NewParentPath>` that is `.class` itself, or that
   passes through an existing `.class/` directory anywhere along the way
   — the same reservation `create` enforces before doing anything else
   (see `create.md`).
3. Computes the destination path: `<NewParentPath>/<old final segment>`.
   Errors if something already exists there — `move` never silently
   overwrites or merges two Nodes.
4. Since a Node's class name is its full path from `./Topology` flattened
   with `_` (see `../Node/nesting.md`), relocating it changes not just
   its own class name but every descendant Node's too — same as `rename`
   (see `rename.md`, steps 2-3): finds every descendant Node nested
   inside `<NodePath>`'s own directory, at any depth, and computes each
   one's new path (and so new flattened class name) by substituting the
   relocated prefix, leaving whatever's nested below each descendant
   unchanged.
5. Scans every Node anywhere under `./Topology` for occurrences that need
   updating, for `<NodePath>` and every descendant found in step 4 alike
   — the same scan `rename` performs (see `rename.md` step 4): each one's
   own class declaration (only inside its own `.class/<own segment>.py`,
   never elsewhere), every transform's `inputs`/`output` bindings and any
   `self.inputs[<old path>]` lookup inside its own `apply()`, and any
   `Agent.free(...)`-authored prose that names one of them — a judgment
   call, not a plain text find/replace, since references can be
   indirect.
6. Creates any missing intermediate segment of `<NewParentPath>` as a
   plain organizational folder (same handling `create` gives intermediate
   segments — see `create.md`), then physically relocates `<NodePath>`'s
   own directory — `.class/`, its own content, and everything nested
   inside it — to the destination computed in step 3. A single move, not
   a delete-then-recreate: nothing about the Node's own `.class/` files
   is regenerated or re-derived, besides the class-name updates in the
   next step.
7. Updates the class declaration inside the moved Node's own (now
   relocated) class file to its new flattened name, and does the same
   inside each descendant's own class file found in step 4 — a content
   edit only, since none of those files themselves get renamed or moved
   independently of their parent (they travel with it on disk
   automatically — see `../Node/nesting.md`'s "File names").
8. Rewrites every occurrence found in step 5 to the corresponding new
   path (for `load_node_module(...)` calls and `self.inputs[...]`
   lookups) or new class name (for a class declaration).

## Purpose

The relocation counterpart to `rename`'s in-place renaming: both exist
because a Node's identity — its class name, and everything nested inside
it — is derived from its full path (see `../Node/nesting.md`), so neither
a name change nor a parent change can ever be "just" a filesystem
operation without also chasing down every reference elsewhere in the
Topology. `move` keeps a Node's own `.class/`-derived logic
(`properties()`, any transforms it hosts) and every other Node's
dependency on it intact across the relocation, the same way `rename`
keeps them intact across a name change — the fix `remove` (`remove.md`)
plus `create` (`create.md`) can't offer, since that pair would discard
the Node's generated logic entirely and force every dependent to be
rewired from scratch. Like `rename`, `move` has no "nothing else may
depend on it" precondition (unlike `remove`): every dependent gets
rewritten instead of being required to already be gone.
