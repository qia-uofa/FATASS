# Agent

`Agent` is the class every command documented under `Command Tree/` is a
method on (see `README.md`'s Commands section). This file documents its
own properties — what every command has access to via `self`, separately
from what each individual command does.

## Properties

### `agent.free(task: str)`

The escape hatch every command falls back to for any step with no clean
procedural translation — a judgment call, open-ended generation, an
error-handling branch that can't be decided deterministically. `task` is
a natural-language description of what to do; the return value is
whatever the caller needs (str, list, dict, ...). Stateless — unlike
`agent.cwd` below, calling it doesn't change anything about `Agent`
itself. Named after the `free` command (see `Command Tree/free.md`),
which *is* a direct call to this method, `self` scoped to a single
Node — every other command's use of it is the same escape hatch, just
reached from inside a more structured command instead of exposed
directly.

### `agent.cwd: Path`

The one piece of persistent state `Agent` carries: a plain relative path
under `./Topology`, initialized to `.` (the root) when the agent starts.
Changed only by `cd` (see `Command Tree/cd.md`) — every other command
only *reads* it.

`init` is what creates `./Topology` itself the very first time it's
missing (see `Command Tree/init.md`); every command still assumes
`./Topology` already exists by the time it runs, `init` included on
every run after the first.

## How commands use `agent.cwd`

Every command that takes a `<NodePath>` or `<TransformPath>` — including
the `<NodePath>` given to `--inputs` in `create` — resolves it relative
to `agent.cwd`, exactly the way a shell resolves a relative path against
its own working directory: join the given path onto `agent.cwd`
(`..`/`.` segments included), then read that as a path under
`./Topology` (see `Node/nesting.md`). Concretely: wherever a command's
Behavior section refers to "resolves `<NodePath>`", read that as "joins
`<NodePath>` onto `agent.cwd`, then resolves the result under
`./Topology`" — never a hardcoded `./Topology` on its own, and never the
Node's class name, even though that name is unique: it spells out the
Node's entire path from `./Topology`, which is exactly what typing a
short relative path instead is meant to avoid (see `Node/nesting.md`'s
"Addressing").

This applies uniformly, `do`'s raw Python line included: the
`node(path)`/`transform(path)` helpers it calls (see `Node/nesting.md`)
are the same `agent.cwd`-relative resolution as every structured
command's own `<NodePath>`/`<TransformPath>` argument — a path typed at
the prompt always means the same thing, however it's spelled into a
command.

The one place this *doesn't* apply is inside generated `.class/*.py`
code itself (a transform's `inputs`/`output`, see `Node/transform.md`):
those are resolved once, at `create` time, into a `./Topology`-relative
form that's baked into the generated source — stable regardless of
wherever `agent.cwd` happens to be later, at the moment `apply()`
actually runs.
