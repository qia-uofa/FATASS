# `apply`

## Syntax

```text
>>> apply <NodeName> <TransformName> [args...]
```

- `<NodeName>` — an existing Node's class name.
- `<TransformName>` — the key one of `<NodeName>`'s registered
  transforms is stored under (see `../Node/transform.md`).
- `[args...]` — optional arguments forwarded to the transform's `apply`
  method, if it requires any.

Shorthand for finding `<NodeName>`'s transform registered under
`<TransformName>` and applying it — equivalent to:

```text
>>> do "<NodeName>().transforms['<TransformName>'].apply(args...)"
```

Unlike `build` (`build.md`), `apply` never purges `<NodeName>`'s content
first — that purge is `build`'s own mechanics for its one reserved
`"build"` key, not something running a transform does in general.

## Behavior

1. Resolves `<NodeName>` against the current `agent.topology` and
   instantiates it, exactly as `do` does (see `do.md`, `../agent.md`).
2. Looks up `transforms['<TransformName>']` on the instance. If the Node
   has no transform registered under that exact key, this is an error —
   surfaced as-is; nothing is touched when this happens.
3. Calls `apply(...)`, passing through `[args...]` if any were given.
   Nothing under `<NodeName>` is purged beforehand.
4. `apply()`'s access is restricted per `../Node/transform.md`: edit
   access only to the Node the transform is registered on
   (`<NodeName>`, as `output`), read-only to its transform's `inputs`,
   and no edit access to any `.class/` directory — `<NodeName>`'s own
   included.
5. Follows the verbatim-command contract like `do`: whatever `apply`
   returns, or raises, is reported back unchanged.

### Example

```text
>>> apply Overview build
```

is equivalent to:

```text
>>> do "Overview().transforms['build'].apply()"
```

just written as a dedicated subcommand instead of a raw Python line.
`build Overview` (see `build.md`) is the special case of `apply Overview
build` that also purges `Overview`'s content first — the one transform
key that gets its own even-shorter subcommand, by convention the one a
Node uses for its primary "regenerate my content" transform.
