# `update`

## Syntax

```text
>>> update [-r] <NodePath> [args...]
```

- `<NodePath>` — an existing Node, relative to `agent.cwd` (see
  `../agent.md`).
- `-r` — before updating this Node itself, recursively `update`s every
  Node named in its `"build"` transform's `inputs` first (their own
  `"build"` inputs, and so on) — see Behavior step 3. Without `-r`, only
  this Node is updated; its inputs are read exactly as they currently
  stand.
- `[args...]` — optional arguments forwarded to this Node's own
  transform's `apply` method, if it requires any. Never forwarded to any
  Node updated recursively via `-r`.

Behaves exactly like `build` (see `build.md`) — same Node resolution,
same `"build"`-key lookup, same recursive `-r` walk over build-inputs —
minus the purge: `update` never empties anything under this Node (or,
with `-r`, any of its build-dependencies') before calling `apply()`.
Equivalent to:

```text
>>> apply <NodePath>/.class/build.py [args...]
```

with `-r` added on top — `apply` has no `-r` of its own.

## Behavior

1. Resolves `<NodePath>` against `agent.cwd` and instantiates it, exactly
   as `build`/`apply`/`do` do (see `build.md`, `apply.md`, `do.md`,
   `../Node/nesting.md`, `../agent.md`).
2. Looks up `transforms['build']` on the instance. If the Node has no
   transform registered under that exact key, this is an error —
   surfaced as-is, not silently redirected to a differently-named
   transform; nothing is touched when this happens.
3. If `-r` was given, resolves each Node named in that transform's
   `inputs` (each stored as a `./Topology`-relative path — see
   `../Node/transform.md`) and recursively runs `update -r` (no
   `args...`) on each one *before* continuing to step 4 for this Node
   itself — same bottom-up, leaves-first order and the same
   already-updated tracking `build -r` uses (see `build.md` step 3), just
   without purging anything at any level. Without `-r`, this step does
   nothing — this Node's inputs are read exactly as they currently
   stand.
4. Calls `apply(...)`, passing through `[args...]` if any were given.
   Nothing under this Node, or any Node updated recursively via `-r`, is
   purged at any point — whatever a transform's `apply()` finds already
   there when it runs is what it runs against.
5. `apply()`'s access is restricted per `../Node/transform.md`: edit
   access only to the Node being updated (as `output`), read-only to its
   transform's `inputs`, and no edit access to any `.class/` directory —
   its own included.
6. Follows the verbatim-command contract like `do`: whatever `apply`
   returns, or raises, is reported back unchanged.

## Purpose

The accumulate-mode counterpart to `build`'s always-regenerate-from-
scratch guarantee (see `build.md`'s Remark): for a transform whose own
logic genuinely needs to build on prior output (e.g. "find the next free
index"), `update` runs it against whatever's already there instead of
forcing a clean slate first. `-r` extends that up the whole
build-dependency graph — plain `update <NodePath>` leaves stale inputs
exactly as they are; `update -r <NodePath>` refreshes them too, still
without purging anything, anywhere.
