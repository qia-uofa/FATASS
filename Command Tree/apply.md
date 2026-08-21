# `apply`

## Syntax

```text
>>> apply <TransformPath> [args...]
```

- `<TransformPath>` — the relative path (from `agent.cwd`, see
  `../agent.md`) to an existing transform's own `.py` file — a Node path
  with `.class/<TransformName>.py` appended (see `../Node/nesting.md`),
  e.g. `MyNode/.class/SpecificTransform.py`.
- `[args...]` — optional arguments forwarded to the transform's `apply`
  method, if it requires any.

Shorthand for resolving `<TransformPath>` and applying it — equivalent
to:

```text
>>> do "transform('<TransformPath>').apply(args...)"
```

Unlike `build` (`build.md`), `apply` never purges the owning Node's
content first — that purge is `build`'s own mechanics for its one
reserved `"build"` key, not something running a transform does in
general.

## Behavior

1. Resolves `<TransformPath>` against `agent.cwd` and instantiates the
   owning Node, exactly as `do` does (see `do.md`, `../Node/nesting.md`,
   `../agent.md`).
2. Looks up `transforms['<TransformName>']` on the instance (the
   transform's own filename, minus `.py`). If the Node has no transform
   registered under that exact key, this is an error — surfaced as-is;
   nothing is touched when this happens.
3. Calls `apply(...)`, passing through `[args...]` if any were given.
   Nothing under the owning Node is purged beforehand.
4. `apply()`'s access is restricted per `../Node/transform.md`: edit
   access only to the Node the transform is registered on (as `output`),
   read-only to its transform's `inputs`, and no edit access to any
   `.class/` directory.
5. Follows the verbatim-command contract like `do`: whatever `apply`
   returns, or raises, is reported back unchanged.

### Example

```text
>>> apply Overview/.class/build.py
```

is equivalent to:

```text
>>> do "transform('Overview/.class/build.py').apply()"
```

just written as a dedicated subcommand instead of a raw Python line.
`build Overview` (see `build.md`) is the special case of
`apply Overview/.class/build.py` that also purges `Overview`'s content
first — the one transform key that gets its own even-shorter subcommand,
by convention the one a Node uses for its primary "regenerate my
content" transform.
