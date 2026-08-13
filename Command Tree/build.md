# `build`

## Syntax

```text
>>> build [-r] <NodeName> [args...]
```

- `<NodeName>` — an existing Node's class name.
- `-r` — before building `<NodeName>` itself, recursively `build`s every
  Node named in its `"build"` transform's `inputs` first (their own
  `"build"` inputs, and so on) — see Behavior step 3. Without `-r`, only
  `<NodeName>` is built; its inputs are read exactly as they currently
  stand.
- `[args...]` — optional arguments forwarded to `<NodeName>`'s own
  transform's `apply` method, if it requires any. Never forwarded to any
  Node built recursively via `-r` — see Behavior step 3.

Shorthand for finding `<NodeName>`'s transform registered under the key
`"build"` and applying it — *almost* equivalent to:

```text
>>> apply <NodeName> build [args...]
```

with one deliberate difference: `build` purges `<NodeName>`'s content
first (see Behavior step 4) — `apply` does not. That's `build`'s own
mechanics, not something any transform's `apply()` has to implement for
itself. `-r` has no `apply`/`do` equivalent; it's `build`-only.

## Behavior

1. Resolves `<NodeName>` against the current `agent.topology` and
   instantiates it, exactly as `apply`/`do` do (see `apply.md`, `do.md`,
   `../agent.md`).
2. Looks up `transforms['build']` on the instance. If the Node has no
   transform registered under that exact key, this is an error — surfaced
   as-is, not silently redirected to a differently-named transform (e.g.
   Materials's `organize`); nothing is purged or otherwise touched when
   this happens.
3. If `-r` was given, resolves each Node named in that transform's
   `inputs` and recursively runs `build -r <InputNode>` (no `args...`)
   on each one *before* continuing to step 4 for `<NodeName>` itself —
   so the whole build-dependency graph (see `graph.md` to visualize it)
   gets rebuilt bottom-up, leaves first. Keeps track of every Node
   already (re)built during this call, so a dependency shared by more
   than one Node (or a circular build-dependency) is built once, not
   once per dependent and never forever. Without `-r`, this step does
   nothing — `<NodeName>`'s inputs are read exactly as they currently
   stand, however stale.
4. Purges `<NodeName>`'s own content — everything under it except
   `.class/` — exactly as `purge <NodeName>` does on its own
   (non-recursively, regardless of this command's own `-r`; see
   `purge.md`), just as an unconditional step here rather than the whole
   command. This guarantees a clean slate on
   every `build`, regardless of what a given transform's own logic would
   otherwise do (e.g. a transform that used to accumulate output across
   runs no longer can when invoked this way — see the Remark below).
5. Calls `apply(...)`, passing through `[args...]` if any were given.
6. `apply()`'s access is restricted per `../Node/transform.md`: edit
   access only to the Node being built (`<NodeName>`, as `output`),
   read-only to its transform's `inputs`, and no edit access to any
   `.class/` directory — `<NodeName>`'s own included.
7. Follows the verbatim-command contract like `do`: whatever `apply`
   returns, or raises, is reported back unchanged.

Remark: this makes `build` unconditionally regenerate-from-scratch, full
stop — there is no accumulate-mode `build` invocation. A transform whose
own logic tries to build on prior output (e.g. "find the next free
index") will simply never see any, since `build` always purges first;
that logic still runs, it just always starts from nothing. If a Node's
transform genuinely needs to accumulate output across runs, it must be
invoked through `update <NodeName>` (see `update.md`) instead of `build`
— `update` behaves identically otherwise (including its own `-r`) but
never purges anything first.

### Example

```text
>>> build Overview
```

first purges `./Topology/Overview/` (except `.class/`), then is
equivalent to:

```text
>>> apply Overview build
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
