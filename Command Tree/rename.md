# `rename`

## Syntax

```text
>>> rename <Path> <NewName>
```

- `<Path>` — the existing Node or Transform to rename, relative to
  `agent.cwd` (see `../agent.md`), using the same name-shape detection
  as `create` (see `create.md`):
  - **Node path** — renames a Node.
  - **Transform path** (ends in `.class/<TransformName>.py`) — renames a
    Transform registered on the owning Node.
- `<NewName>` — the replacement: a bare, valid Python class name, not a
  path. `rename` changes a Node's own folder name (or a Transform's own
  filename) in place; it doesn't relocate anything to a different
  parent — that's `move` (`move.md`), for a Node, which handles exactly
  that relocation without discarding the Node's `.class/`-derived logic
  the way a `remove` plus `create` would.

## Behavior

### Renaming a Node (`<Path>` is a Node path)

1. Resolves `<Path>` against `agent.cwd` (see `../Node/nesting.md`,
   `../agent.md`), same as `do`/`apply`/`build`/`update`.
2. Computes this Node's new `./Topology`-relative path: the same parent,
   final segment replaced with `<NewName>`. Since a Node's class name is
   its full path flattened with `_` (see `../Node/nesting.md`), this
   also changes its class name — from the old path flattened, to the
   new one.
3. Finds every descendant Node nested inside this Node's own directory,
   at any depth (see `../Node/nesting.md`) — each one's class name
   changes too, since it embeds this Node's own path as a prefix. For
   each, computes its own new path (and so new class name) by
   substituting the renamed prefix, leaving whatever's nested below this
   Node itself unchanged. A descendant's `.class/` *file* names are
   unaffected either way — they're always just that descendant's own
   final segment (see `../Node/nesting.md`'s "File names"), never the
   flattened form, so nothing about them needs renaming on disk.
4. Scans every Node anywhere under `./Topology` for occurrences that
   need updating, for this Node and every descendant found in step 3
   alike: each one's own class declaration (`class <old flattened>
   (Node):` — only inside its *own* `.class/<own segment>.py`, found via
   step 3, never elsewhere), every transform's `inputs`/`output` and any
   `self.inputs['<old path>']` lookup inside its own `apply()` (both use
   the plain, unflattened path — see `../Node/transform.md`), and any
   `Agent.free(...)`-authored prose that names one of them — a judgment
   call, not a plain text find/replace, since references can be
   indirect.
5. Renames this Node's own directory (its final path segment) to
   `<NewName>`, and its class and description files (see
   `../Node/class.md`) to match — since every descendant's directory
   sits inside this one, they move along with it on disk automatically,
   with no filename changes of their own (step 3). Updates the class
   declaration inside this Node's own (now-renamed) class file to its
   new flattened name, and does the same inside each descendant's own
   class file found in step 3 — a content edit only, since none of
   those files themselves get renamed or moved independently.
6. Rewrites every occurrence found in step 4 to the corresponding new
   path (for `load_node_module(...)` calls and `self.inputs[...]`
   lookups) or new class name (for a class declaration).

### Renaming a Transform (`<Path>` is a Transform path)

1. Resolves the owning Node and confirms `<TransformName>` (`<Path>`'s
   final segment, minus `.py`) is one of its registered transforms (same
   resolution as `do`).
2. Scans every Node anywhere under `./Topology`'s transform modules for
   occurrences of `<TransformName>` that need updating: the owning
   Node's `transform_names()` list, and any other transform that invokes
   `transforms['<TransformName>']` on it (same judgment-call scan
   `remove` uses to check for dependents — see `remove.md`).
3. Renames the transform's `.py` file (updating the class name inside
   it) and its `.md` description file to `<NewName>`.
4. Rewrites every occurrence found in step 2 to `<NewName>`.

## Purpose

Keeps a Node or Transform's name consistent everywhere it's referenced
— its own `.class/` files, every other Node's `inputs`/`output`
bindings, `transform_names()` entries, and description files — so a
rename never leaves a stale reference behind the way manually editing
`./Topology` would. Since a Node's class name is derived from its path
(see `../Node/nesting.md`), renaming a Node necessarily also renames
every Node nested inside it — there's no way to change one segment of a
path without changing every flattened name built on top of it, even
though the *files* those classes live in never need renaming themselves
(only their own content). Unlike `remove`, `rename` has no "nothing else
may depend on it" precondition: every dependent gets rewritten instead
of being required to already be gone.
