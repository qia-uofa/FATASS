# `build`

## Syntax

```text
>>> build [-r] <NodePath> [args...]
```

- `<NodePath>` — an existing Node, relative to `agent.cwd` (see
  `../agent.md`).
- `-r` — before building this Node itself, recursively `build`s every
  Node named in its `"build"` transform's `inputs` first (their own
  `"build"` inputs, and so on) — see Behavior step 3. Without `-r`, only
  this Node is built; its inputs are read exactly as they currently
  stand.
- `[args...]` — optional arguments forwarded to this Node's own
  transform's `apply` method, if it requires any. Never forwarded to any
  Node built recursively via `-r` — see Behavior step 3.

Shorthand for finding this Node's transform registered under the key
`"build"` and applying it — *almost* equivalent to:

```text
>>> apply <NodePath>/.class/build.py [args...]
```

with one deliberate difference: `build` purges the Node's content first
(see Behavior step 4) — `apply` does not. That's `build`'s own
mechanics, not something any transform's `apply()` has to implement for
itself. `-r` has no `apply`/`do` equivalent; it's `build`-only.

## Behavior

1. Resolves `<NodePath>` against `agent.cwd` and instantiates it, exactly
   as `apply`/`do` do (see `apply.md`, `do.md`, `../Node/nesting.md`,
   `../agent.md`).
2. Looks up `transforms['build']` on the instance. If the Node has no
   transform registered under that exact key, this is an error — surfaced
   as-is, not silently redirected to a differently-named transform (e.g.
   Materials's `organize`); nothing is purged or otherwise touched when
   this happens.
3. If `-r` was given, resolves each Node named in that transform's
   `inputs` (each stored as a `./Topology`-relative path — see
   `../Node/transform.md`) and, for each one that itself has a `"build"`
   transform, recursively runs `build -r` (no `args...`) on it *before*
   continuing to step 4 for this Node itself — so the whole
   build-dependency graph (see `graph.md` to visualize it) gets rebuilt
   bottom-up, leaves first. An input with no `"build"` transform (a leaf
   Node populated by hand or via `free` — e.g. raw source material) is
   left untouched rather than erroring: `-r` only rebuilds what's
   rebuildable, and reads everything else exactly as it currently
   stands. Keeps track of every Node already (re)built during this
   call, so a dependency shared by more than one Node (or a circular
   build-dependency) is built once, not once per dependent and never
   forever. Without `-r`, this step does
   nothing — this Node's inputs are read exactly as they currently
   stand, however stale.
4. Purges this Node's own content — everything under it except `.class/`
   — exactly as `purge <NodePath>` does on its own (non-recursively,
   regardless of this command's own `-r`; see `purge.md`), just as an
   unconditional step here rather than the whole command. This
   guarantees a clean slate on every `build`, regardless of what a given
   transform's own logic would otherwise do (e.g. a transform that used
   to accumulate output across runs no longer can when invoked this way
   — see the Remark below).
5. Calls `apply(...)`, passing through `[args...]` if any were given.
6. `apply()`'s access is restricted per `../Node/transform.md`: edit
   access only to the Node being built (as `output`), read-only to its
   transform's `inputs`, and no edit access to any `.class/` directory —
   its own included.
7. Follows the verbatim-command contract like `do`: whatever `apply`
   returns, or raises, is reported back unchanged.

Remark: this makes `build` unconditionally regenerate-from-scratch, full
stop — there is no accumulate-mode `build` invocation. A transform whose
own logic tries to build on prior output (e.g. "find the next free
index") will simply never see any, since `build` always purges first;
that logic still runs, it just always starts from nothing. If a Node's
transform genuinely needs to accumulate output across runs, it must be
invoked through `update <NodePath>` (see `update.md`) instead of `build`
— `update` behaves identically otherwise (including its own `-r`) but
never purges anything first.

### Example

```text
>>> build Overview
```

first purges `./Topology/Overview/` (except `.class/`), then is
equivalent to:

```text
>>> apply Overview/.class/build.py
```

Given `Overview`'s `"build"` transform reads from `Materials` and
`Notes`:

```text
>>> build -r Overview
```

first (recursively) builds `Materials` and `Notes` — purging and
re-applying each in turn, along with *their* own build-inputs, if any —
then purges and builds `Overview` itself from the now-freshly-built
result.

This exists because `build` is, by convention, the transform name a Node
uses for its primary "regenerate my content" transform — a dedicated,
even-shorter subcommand for that one common case, alongside the
general-purpose `apply` (and the fully general `do`).
