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
`agent.topology` below, calling it doesn't change anything about `Agent`
itself. Named after the `free` command (see `Command Tree/free.md`),
which *is* a direct call to this method, `self` scoped to a single
Node — every other command's use of it is the same escape hatch, just
reached from inside a more structured command instead of exposed
directly.

### `agent.topology: Topology`

The one piece of persistent state `Agent` carries: which `Topology` (see
`Node/topology.md`) the agent is currently working under. Initialized to
the root Topology (reserved class name `topology`) when the agent
starts. Changed only by `cd` (see `Command Tree/cd.md`) — every other
command only *reads* it, never writes it.

Instantiating the root this way requires `./.class/topology.py` — at the
outer repo's own root, sibling to `./Topology` (see `Node/topology.md`)
— to already exist. That's exactly what `init`'s bootstrap step
guarantees, the first time it ever runs against a repo that doesn't have
one yet (see `Command Tree/init.md`). In a brand new working directory,
before `init` has run once, there's nothing yet for `agent.topology` to
actually load: `init` is the one command that tolerates this and is
allowed to run without a fully-resolved `agent.topology` — its whole
job on that first run is to make one resolvable. Every other command
assumes `agent.topology` already points at a real, loadable Topology
instance, root or otherwise.

## How commands use `agent.topology`

Every command documented under `Command Tree/` that resolves a
`<NodeName>` — directly, or as the `<NodeName>` half of a `<Name>` in
`create`/`remove`/`rename`'s dotted shape (see `Command Tree/create.md`)
— does so against `agent.topology.topology_path`, never against a
hardcoded `./Topology`. Concretely: wherever a command's Behavior
section refers to `./Topology/<NodeName>/...`, read that as
`agent.topology.topology_path/<NodeName>/...` — the literal `./Topology`
root is only what it resolves to before any `cd` has run, or after
`cd ,` returns to it (see `Command Tree/cd.md`).

This means the *same* command means something different depending on
`agent.topology`: `>>> do "Notes().properties()"` resolves `Notes`
against whatever Topology is current, not always the repo root. A
command never reaches outside `agent.topology.topology_path` on its own
initiative — moving to a different Topology is always an explicit,
separate `cd` first.
